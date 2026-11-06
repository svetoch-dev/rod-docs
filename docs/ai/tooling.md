# Bazel tooling context path

1. Read the consumer's `MODULE.bazel` for the pinned Bazel modules, extensions, repositories, and tool versions.
2. Read the affected `BUILD.bazel` to see which external labels are used. For helper binaries, follow the extension that provides the executable; for Python targets, follow the `pip.parse` lock and imported requirements; for images, follow the OCI rule and any shared image macro.
3. Inspect the selected rule or macro at the pinned ref before changing inputs, generated files, or targets. Dependencies declared in `MODULE.bazel` alone do not establish that a particular target uses them.

The consuming repository owns its tool versions, image inputs, and CI target selection.
