# video-swift-ui


# Play a video once, then loop its last N seconds (SwiftUI + AVFoundation)

**Requirement:** play a full video, then loop its final N seconds (e.g. 5) with no visible seam until dismissed. Show an X button top-right that dismisses it.

**Approach**
- Use an `AVQueuePlayer`. Its first item is the full video.
- On iOS 17+, create `AVPlayerLooper(player:templateItem:timeRange:existingItemsOrdering: .loopingItemsFollowExistingItems)` with `timeRange = [duration − N s, duration]`. The looper keeps pre-buffered copies of the tail queued behind the full video, so nothing is seeked at the loop point.
- On iOS 16, the old looper init puts loop items *ahead* of existing items. Fallback: turn the tail into its own `AVMutableComposition` that starts at 0, and keep two copies queued ("treadmill").
- SwiftUI: playback lives in an `ObservableObject` held by `@StateObject`; start it in `.task`, stop it in `.onDisappear`. `VideoPlayer` always shows system controls, so a tiny `UIViewRepresentable` wrapping `AVPlayerLayer` is the video surface. The close button is a SwiftUI overlay.
- Keep a strong reference to the looper. Call `disableLooping()` and `removeAllItems()` on dismiss.
- Load `duration` asynchronously before creating the looper.
- If the loop seam looks wrong, check the content: the frame at (end − N s) must visually follow the last frame.

```swift
import AVFoundation
import SwiftUI
import UIKit

// Plays a video once, then seamlessly loops its final `loopSeconds` until it is
// dismissed (X button) or the view goes away.
//
// Usage:
//
//     .fullScreenCover(isPresented: $showVideo) {
//         VideoView(videoURL: Bundle.main.url(forResource: "video", withExtension: "mp4")!) {
//             showVideo = false
//         }
//     }
//
// To dismiss it from elsewhere, set `showVideo = false` (or drive it from your app
// state). The view disappearing stops playback.

// MARK: - View

struct VideoView: View {
    private let videoURL: URL
    private let loopSeconds: Double
    private let videoGravity: AVLayerVideoGravity
    private let onClose: () -> Void

    @StateObject private var playback = VideoPlayback()

    /// - Parameters:
    ///   - videoGravity: `.resizeAspect` shows the whole video; `.resizeAspectFill` fills the screen.
    init(videoURL: URL,
         loopSeconds: Double = 5,
         videoGravity: AVLayerVideoGravity = .resizeAspect,
         onClose: @escaping () -> Void) {
        self.videoURL = videoURL
        self.loopSeconds = loopSeconds
        self.videoGravity = videoGravity
        self.onClose = onClose
    }

    var body: some View {
        ZStack(alignment: .topTrailing) {
            Color.black.ignoresSafeArea()

            PlayerLayerView(player: playback.player, videoGravity: videoGravity)
                .ignoresSafeArea()

            Button {
                playback.stop()
                onClose()
            } label: {
                Image(systemName: "xmark")
                    .font(.system(size: 15, weight: .bold))
                    .foregroundStyle(.white)
                    .frame(width: 44, height: 44)
                    .background(.black.opacity(0.55), in: Circle())
            }
            .accessibilityLabel("Close")
            .padding(12)
        }
        .task { await playback.start(url: videoURL, loopSeconds: loopSeconds) }
        .onDisappear { playback.stop() }
        // AVPlayer pauses when the app goes to the background; resume on return.
        .onReceive(NotificationCenter.default.publisher(for: UIApplication.willEnterForegroundNotification)) { _ in
            playback.resume()
        }
    }
}

// MARK: - Playback model

/// Owns the player. Plays the full video once, then loops its last N seconds.
///
/// How it works: an `AVQueuePlayer` holds the full video as its first item, and an
/// `AVPlayerLooper` appends pre-buffered copies of the last N seconds *behind* it.
/// The next copy is always loaded before the current one ends, so there is no seek
/// and no visible gap at the loop point.
@MainActor
final class VideoPlayback: ObservableObject {
    let player = AVQueuePlayer()

    private var looper: AVPlayerLooper?               // iOS 17+. Must be retained or looping stops.
    private var loopAsset: AVAsset?                    // iOS 16 fallback only.
    private var ownItems = Set<ObjectIdentifier>()     // iOS 16 fallback: items this player queued.
    private var endObserver: NSObjectProtocol?
    private var hasStarted = false
    private var isStopped = false

    func start(url: URL, loopSeconds: Double) async {
        guard !hasStarted, !isStopped else { return }
        hasStarted = true

        let asset = AVURLAsset(url: url)
        do {
            // Load the duration up front (AVPlayerLooper requires it, and it avoids blocking the main thread).
            let duration = try await asset.load(.duration)
            guard !isStopped else { return }

            let loopLength = CMTime(seconds: loopSeconds, preferredTimescale: 600)
            let loopStart = CMTimeMaximum(.zero, duration - loopLength)
            let loopRange = CMTimeRange(start: loopStart, end: duration)

            // 1. The full video plays first.
            let intro = AVPlayerItem(asset: asset)
            ownItems.insert(ObjectIdentifier(intro))
            player.insert(intro, after: nil)

            // 2. The final N seconds loop forever, queued behind it.
            if #available(iOS 17.0, *) {
                looper = AVPlayerLooper(
                    player: player,
                    templateItem: AVPlayerItem(asset: asset),
                    timeRange: loopRange,
                    existingItemsOrdering: .loopingItemsFollowExistingItems
                )
            } else {
                try startManualLoop(asset: asset, loopRange: loopRange)
            }

            player.play()
        } catch {
            // The close button stays visible, so the user can still dismiss the view.
            print("Video failed to load: \(error)")
        }
    }

    func resume() {
        guard hasStarted, !isStopped else { return }
        player.play()
    }

    /// Stops playback and looping. Safe to call more than once.
    func stop() {
        guard !isStopped else { return }
        isStopped = true
        if let endObserver { NotificationCenter.default.removeObserver(endObserver) }
        endObserver = nil
        looper?.disableLooping()
        looper = nil
        player.pause()
        player.removeAllItems()
        ownItems.removeAll()
    }

    /// iOS 16 fallback. Before iOS 17, AVPlayerLooper always puts its loop *ahead* of
    /// existing queue items, so we run a small "treadmill" instead: the loop segment
    /// becomes its own asset that starts at 0 (no seeking), and two copies are always
    /// queued so the next one is preloaded. Delete this if your minimum target is iOS 17.
    private func startManualLoop(asset: AVAsset, loopRange: CMTimeRange) throws {
        let segment = AVMutableComposition()
        try segment.insertTimeRange(loopRange, of: asset, at: .zero)
        loopAsset = segment

        enqueueLoopSegment()
        enqueueLoopSegment()

        endObserver = NotificationCenter.default.addObserver(
            forName: AVPlayerItem.didPlayToEndTimeNotification, object: nil, queue: .main
        ) { [weak self] note in
            MainActor.assumeIsolated {
                // Only react to items this player queued (ignore other players in the app).
                guard let self, let item = note.object as? AVPlayerItem,
                      self.ownItems.remove(ObjectIdentifier(item)) != nil else { return }
                self.enqueueLoopSegment()
            }
        }
    }

    private func enqueueLoopSegment() {
        guard let loopAsset, !isStopped else { return }
        let item = AVPlayerItem(asset: loopAsset)
        ownItems.insert(ObjectIdentifier(item))
        player.insert(item, after: nil)
    }
}

// MARK: - Video surface

/// SwiftUI's `VideoPlayer` always shows the system playback controls (play/pause,
/// scrubber), which we don't want here. This small bridge shows a bare AVPlayerLayer
/// instead. It's the standard way to show control-free video in SwiftUI.
private struct PlayerLayerView: UIViewRepresentable {
    let player: AVPlayer
    let videoGravity: AVLayerVideoGravity

    func makeUIView(context: Context) -> PlayerUIView {
        let view = PlayerUIView()
        view.playerLayer.player = player
        view.playerLayer.videoGravity = videoGravity
        return view
    }

    func updateUIView(_ view: PlayerUIView, context: Context) {
        view.playerLayer.player = player
        view.playerLayer.videoGravity = videoGravity
    }

    final class PlayerUIView: UIView {
        override class var layerClass: AnyClass { AVPlayerLayer.self }
        var playerLayer: AVPlayerLayer { layer as! AVPlayerLayer }
    }
}

// MARK: - Preview

#Preview {
    if let url = Bundle.main.url(forResource: "video", withExtension: "mp4") {
        VideoView(videoURL: url) {}
    } else {
        Text("Add video.mp4 to the app target to preview")
    }
}
```
