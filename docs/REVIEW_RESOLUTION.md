# Review resolution contract

This addendum is a documentation-only acceptance contract for PR #9. It records the resolution required for each existing review thread. It does not claim that product implementation or tests have been completed. The existing Bot review is the sole Bot input for this PR and will not be retriggered.

## PRRT_kwDOTN39ls6OYqix — CSP directives are explicit

Finding: The security baseline must make the default-deny policy and every permitted resource class auditable.

Normative resolution:
- The generated document and serving mode must define an explicit default-deny Content-Security-Policy.
- The policy must explicitly account for script-src, style-src, worker-src, form-action, base-uri, and object-src; omitted directives must not silently inherit an unsafe default.
- Inline script, inline style, eval-like execution, object/plugin loading, unrestricted form submission, and uncontrolled worker creation remain denied unless a separately reviewed requirement adds a narrowly scoped source.
- The design must not claim a stronger policy than the actual emitted headers or static metadata.

Focused verification before resolving this thread:
- Inspect the emitted policy for every supported serving mode and assert the required directives and deny-by-default behavior.
- Exercise negative cases for inline/eval-like script, inline style, object loading, arbitrary form targets, and unauthorized worker sources.
- Confirm the documented policy and the implementation acceptance criteria describe the same source set.

## PRRT_kwDOTN39ls6OYqiz — export dimensions and detail sizes are integral

Finding: Fixed export sizes of 512 and 1024 do not safely pair with every proposed detail size of 48 and 96.

Normative resolution:
- The valid detail-size set must be constrained to divisors of the selected export size, or the working/export dimensions must be derived so the relationship is exact.
- A non-divisible request must be rejected with a clear validation error; it must not silently blur, crop, pad, or change resolution.
- The design must state which dimension is canonical when a user selects a preset and how the invalid combination is reported.

Focused verification before resolving this thread:
- Run a matrix covering export sizes 512 and 1024 and detail sizes 48 and 96, including every accepted and rejected pair.
- Assert that accepted pairs preserve exact integer geometry and rejected pairs produce no altered image output.
- Check that preset documentation and CLI/config validation use the same divisibility rule.

## PRRT_kwDOTN39ls6OYqi2 — PNG privacy metadata is scoped precisely

Finding: PNG output can contain benign resolution metadata, so a blanket claim that all metadata is absent is not an accurate privacy requirement.

Normative resolution:
- The privacy gate must reject EXIF, GPS, and other privacy-bearing ancillary chunks identified by the format policy.
- Benign resolution metadata may remain when it is not privacy-bearing and is explicitly allowed by the output policy.
- The documentation must describe the tested forbidden-chunk/field set rather than claiming universal metadata removal.

Focused verification before resolving this thread:
- Inspect representative PNG binaries for forbidden EXIF/GPS/privacy chunks and fail when any prohibited item is present.
- Include an allowed benign resolution metadata case and confirm it is not misclassified as location/device data.
- Confirm the privacy checklist and output encoder acceptance criteria use the same allow/deny list.

## PRRT_kwDOTN39ls6OYqi4 — dithering occurs during palette selection

Finding: Applying dithering after palette assignment cannot correct the palette decision and is ineffective for the intended quantization quality.

Normative resolution:
- Bayer or error-diffusion dithering must participate in color quantization/palette selection, before or within the mapping from source pixels to palette entries.
- A post-assignment pass that only changes already-selected palette indices is not an acceptable implementation of this requirement.
- The design must identify edge handling, deterministic seed/order, and the no-dither path so output remains reproducible.

Focused verification before resolving this thread:
- Use a gradient and flat-color fixture to compare a deterministic dithered result with the no-dither result.
- Assert that palette selection incorporates the propagated/thresholded error and that the output uses only the declared palette.
- Repeat the same input to verify byte-identical deterministic output.

## PRRT_kwDOTN39ls6OYqi6 — quality coverage enumerates all six presets

Finding: The product declares six shipped presets but the quality spike only covers five.

Normative resolution:
- The canonical preset list must contain all six shipped presets, with names, dimensions, detail parameters, and quality expectations defined in one source.
- The quality spike, fixtures, and acceptance matrix must enumerate that same six-item list; an omitted preset is a coverage failure.
- A preset must not be called shipped until its documented quality and geometry checks are represented.

Focused verification before resolving this thread:
- Compare the canonical preset registry against the quality-spike matrix and require exact set equality.
- Run the acceptance checklist for each of the six entries, including the smallest and largest dimensions.
- Confirm unknown or misspelled preset names fail validation instead of selecting a fallback.

## PRRT_kwDOTN39ls6OYqi8 — crop precedes resize and preserves source data

Finding: Cropping a resized working image can lose source detail and make crop coordinates ambiguous.

Normative resolution:
- Preserve the original/decoded source for crop operations.
- Apply the requested crop in source coordinates first, then resize the cropped working region to the selected output/detail dimensions.
- Define behavior for bounds, empty crops, aspect-ratio mismatch, and coordinate rounding; no silent crop or resolution change is allowed.

Focused verification before resolving this thread:
- Use a source fixture with identifiable corner markers and assert that crop selection is evaluated before resizing.
- Test exact-boundary, out-of-bounds, empty, and non-integer coordinate inputs with explicit success/error outcomes.
- Confirm the original source remains available for repeated crops and that the final dimensions match the selected valid preset.

## Scope and review boundary

This file is a design/acceptance contract only. It is not evidence that the implementation or focused checks have already passed. After the relevant implementation and validation evidence exists, each mapped existing thread may be replied to and resolved individually. No Bot review will be triggered again.