# peblo-story-buddy

Peblo Story Buddy is an interactive, multimedia storytelling and educational quiz application built using the Flutter framework. The app creates an engaging audio-narrative experience for young learners, which automatically transitions into a data-driven quiz interface upon story completion.

# Key Architectural Points

 1. Framework Choice & Why
I selected **Flutter (Dart)** for this project due to specific runtime performance and structural benefits:
 **Direct Canvas Rendering:** Flutter's rendering engine ensures smooth, jank-free 60FPS animations and UI transitions, even on mid-range target Android devices.
 **Lightweight State Management:** Using the **Provider** pattern allowed me to strictly isolate business logic (audio handling, lifecycle states) from the structural rendering layer.

 2. Audio-to-Quiz State Transition
When the native Text-to-Speech (`TTSService`) engine signals completion via its background channel handler, performing a direct UI state change can conflict with active frame drawing passes. 
To ensure absolute thread safety and eliminate layout freezes, I deferred the state transition using a framework frame scheduler interceptor:
 dart
  tts.flutterTts.setCompletionHandler(() {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      Provider.of<StoryProvider>(context, listen: false).showQuiz();
    });
  });

 4. Data-Driven Quiz Architecture
The quiz screen layout is completely content-agnostic. Rather than relying on rigid, hardcoded layouts, it uses a modular QuizModel.fromJson() factory layer. The options are generated dynamically using collection mappings. Whether a quiz configuration contains 2 choices or 6 choices, the viewport adjusts automatically without throwing any clipping or layout overflow errors.

 5. Caching Approach
Since the current prototype leverages on-device local TTS engines, memory footprint remains minimal. For future cloud integrations, I designed an extension architecture to read from getApplicationDocumentsDirectory(). It will verify cryptographic local file hashes before making remote URI network lookups, caching downloaded byte buffers to disk to eliminate repeated bandwidth costs.

 6. Audio Loading & Failure States
The entire app lifecycle is managed safely through a dedicated enum state machine (idle, loading, playing, quiz, success, error). If a system audio driver fails to initialize or is interrupted, the exception is instantly caught, updating the interface to a structured failure screen instead of triggering an unhandled app crash.

 7. Performance Profiling
What I Measured: I used Flutter DevTools Performance overlays to actively track frame render timings, GPU/CPU thread execution bottlenecks, and runtime heap allocations during state transitions.
Before/After: Initially, shifting states directly inside the native audio callback triggered immediate frame layout drops and visible transition stutter. Introducing addPostFrameCallback along with structuring presentation elements into cached const constructors flattened processing spikes, keeping frame rendering well below the 16ms threshold.

 8. Device Optimization
To minimize garbage collection frequency on low-to-mid-spec devices, resource-heavy background animation triggers are tightly synchronized to the interface component lifecycles:
@override
void dispose() {
  confetti.dispose(); // Instantly tears down background rendering loop loops when the view is closed
  super.dispose();
}

 9. AI Usage & Judgment
Where it helped: AI was used as a pair-programmer to quickly generate structural data models and lay down boilerplate configuration methods for the native platform Text-to-Speech hooks.

What didn't work: When handling the automatic transition from the story finishing to the quiz rendering, the screen completely froze. The AI continuously suggested structuring asynchronous execution loops inside a global async microtask loop, which did not resolve the underlying issue. The real solution came from identifying the framework thread clash and applying a native addPostFrameCallback scheduler instead.

Rejected Suggestion & Why: The AI initially recommended initializing and storing the ConfettiController animation instances directly inside the global StoryProvider class file to keep the variables grouped together. I completely rejected this suggestion. Animation handles are strictly linked to layout trees and tick rates. Storing them globally breaks clean separation of concerns and creates severe background memory leaks, as the framework loses the ability to dispose of the animation loops out of RAM when the target layout widget dismounts.
