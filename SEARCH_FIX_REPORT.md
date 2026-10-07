# Search/index debugging report

## Diagnosis from the original source

The active pipeline was MainActivity → GallerySyncManager → MediaStore and Room v8 → OcrProcessor → Room → SearchEngine → DuplicateDetector → ImageAdapter. GalleryManager is an older, currently unused alternative. OCR used real ML Kit text recognition. VisualFeatureExtractor stored dominant colour, a 64-bin RGB histogram, grayscale and edges; VisualSimilarity stored a handcrafted grid descriptor. DuplicateDetector stored a 64-bit dHash.

ObjectDetector was not ML: warm pixels generated Pizza/Food, skin-colour pixels generated Person, centre colour/edges generated Dress/Clothing/Shoes, and edges generated Document/Timetable. There was no Dog, Cat or Bird recognition. The detected_objects column persisted those guesses and fixed pseudo-confidence values. Raising/lowering search thresholds could not supply missing dog evidence.

SemanticSearch used positive character/word hashes in only 96 shared dimensions plus four hand-written concept clusters. It was not a learned semantic model. Hash collisions, heuristic labels included in its input, and a query-independent visual baseline admitted unrelated images. Object evidence was capped at 0.25 and weighted at 0.05; object explanations required >0.25 and were therefore unreachable. OCR used substring matches. RGB colour bins overlapped, and the red histogram shortcut bypassed the skin-tone-aware dominant-colour classification.

The current search loop already ran in bgExecutor. Confirmed UI blocking paths were all-gallery quadratic duplicate grouping, filtering/sorting, and synchronous setImageURI decoding. No historical ANR trace or older threshold revisions were supplied; their exact causes cannot be established from this checkout. Zero-result paths include absent object evidence, unchanged rows never retried for missing stages, lack of plural normalization, accidental substring date filters and reintroducing a date-only query as text. The original final result cap was 150.

Room used destructive migration fallback. A failed/null MediaStore query could be interpreted as an empty gallery and erase cached rows. Processing failures could be marked complete. These were separate index integrity problems.

## Implementation and changed files

All Java files below are in app/src/main/java/com/example/finalproject/.

| File | Change |
|---|---|
| app/build.gradle.kts | Bundle ML Kit image-labeling 17.0.9, Glide 4.16.0; enable BuildConfig for debug-only counters. |
| ObjectDetector.java | Replace pixel rules with an actual ML Kit ImageLabeler; preserve returned confidence values; reject invalid serialized scores; no inference during search. |
| ScreenshotEntity.java | Add label_version with default 0 and carry it through model conversions; retain detected_objects for label/confidence storage. |
| ImageModel.java | Carry labelVersion in memory. |
| AppDatabase.java | Version 9 with an explicit 8→9 migration. Preserve rows, OCR, colours, hashes and history; clear only heuristic labels and their completion flag. Remove destructive fallback. |
| GallerySyncManager.java | Queue unchanged images with missing/outdated labels or unfinished processing; keep cached metadata; expose a snapshot while indexing; protect the cache against query failures; debug index counts and shutdown handling. |
| GalleryManager.java | Preserve label version when restoring cached records through the older gallery path. |
| OcrProcessor.java | Run the reusable labeler in the serial indexing executor; save successful labels/version per image; leave failures retryable; retain existing completed OCR/features; stop at an image boundary on close and close native resources after work. Stop generating unused hashed embeddings. |
| SearchEngine.java | Rank actual stored labels, whole-token OCR/filename matches, explicit OCR concepts and requested colours. Normalize plurals/synonyms; retain partial matches; remove hashed/texture qualification, global score cutoff and result cap. Copy result models, use evidence-based explanations, and log counts/timing without private text. Tighten blue/green bins and prevent the red histogram override. |
| SemanticSearch.java | Add explicit OCR concept matching for payment/chat/schedule. Legacy hash helpers remain for compatibility but are not used by live search or indexing. |
| DateQueryParser.java | Require whole-word day/month matches to avoid false date filters inside other words. |
| MainActivity.java | Read Room and search on the executor; move filtering, sorting and duplicate grouping off the UI thread; use snapshots and generation checks to avoid stale result publication; guard destroyed activity callbacks. |
| DuplicateDetector.java | Compare equal-size files for conservative search deduplication, bucket candidates by file size, and retain the strongest ranked representative. Avoid hiding distinct photos solely because their coarse hashes are near each other. Existing duplicate-browsing behaviour is retained. |
| ImageAdapter.java | Load bounded thumbnails asynchronously through Glide with recycled-view handling. |
| ImageViewerActivity.java | Load full-view images asynchronously through Glide. |
| app/src/test/java/com/example/finalproject/SearchRegressionTest.java | Query normalization, date parsing, confidence validation, whole-label matching and OCR concept regressions. |
| app/src/androidTest/java/com/example/finalproject/SearchIndexIntegrationTest.java | Requested query matrix, label-vs-colour ordering, old-heuristic rejection, Room migration/round-trip, actual pixel labeling, background backfill and 7,248-record search checks. |
| app/src/androidTest/assets/dog.jpg | Public TensorFlow example fixture, used only in the test APK. |

ScreenshotDao, layouts and resources did not need changes. Its existing insert/read methods persist the new Room field through generated code.

## Recognition, storage and backfill

The model is bundled, runs on-device, and does not upload gallery images. Reference: https://developers.google.com/ml-kit/vision/image-labeling/android

detected_objects now contains real label/confidence pairs, for example Dog:0.9839. label_version=1 plus is_object_processed=true identifies current successful inference, including successful empty label sets. The model output floor is 0.10 to retain weak candidates; scores rank confidence rather than treating all labels equally. Low-confidence predictions can still be false positives.

Opening/resuming the app automatically runs the existing gallery sync, detects version-0/missing labels and processes those images in the background. No manual image selection or database reset is needed. Each completed image is saved immediately. If the process exits, later gallery sync resumes unfinished rows. This uses the existing app-lifetime indexing pipeline; it is not a new persistent OS background service. Search reads stored Room metadata, so it can use completed backfill rows while other images are still processing.

Dog=0.96 ranks above Dog=0.15. Colour alone qualifies only for a query that asks for colour, and is a much weaker partial match in a mixed query. OCR and label evidence do not both need to exist. Explanations name only contributing sources: Dog requires a current stored label; OCR and Filename are distinguished; Blue requires colour evidence. Colours remain image-level evidence, not proof that a specific object is blue.

## Validation

Final command: `gradlew.bat :app:assembleDebug :app:testDebugUnitTest :app:connectedDebugAndroidTest :app:lintDebug --no-configuration-cache --console=plain -Dhttps.protocols=TLSv1.2` using Android Studio's JBR.

- Debug APK build: passed.
- Unit tests: 5 passed (4 new regressions plus the existing test).
- Emulator instrumentation: 6 passed (5 integration tests plus the existing test).
- Room 8→9 migration: runtime schema validation passed; OCR/hash/colour/history retention and label round-trip passed.
- Real Labrador fixture: Dog confidence 0.9839127659797668; dog search ranked the real labeled fixture first. Background relabeling of an existing row preserved its OCR/hash and persisted the new Dog label.
- Query matrix: dog, dogs, my dog, dog photo, blue dog, cat, bird, red, blue, dress, UPI, Myntra, UPI Rahul, blue dress passed against controlled stored metadata.
- 7,248 synthetic metadata records with 350 dog labels: all 350 returned in 1,163 ms on the emulator. This measures ranking, not model accuracy on 7,248 real images.
- Lint: 0 errors, 24 warnings (dependency/version suggestions, existing resources/plurals, notifyDataSetChanged and version-catalog suggestions).

The phone gallery was not connected. Retrieval accuracy/recall on the user's 300–350 dog photographs, full-gallery indexing time, and the historical ANR remain unverified. Install this build over the existing app without clearing its data, allow the automatic label backfill to finish, and check the debug index counters and dog results on the phone. Existing databases older than v8 are not covered by the supplied migration; no destructive fallback is enabled.

APK: app/build/outputs/apk/debug/app-debug.apk

Test image source: https://storage.googleapis.com/download.tensorflow.org/example_images/YellowLabradorLooking_new.jpg (TensorFlow-hosted example; not part of the application APK).
