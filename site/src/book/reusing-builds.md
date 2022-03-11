# Reusing builds

* **Introduced in:** cargo-nextest 0.9.10
* **Environment variable**: `NEXTEST_EXPERIMENTAL_REUSE_BUILD=1`
* **Tracking issue**: [#98]

[#98]: https://github.com/nextest-rs/nextest/issues/98

In some cases, it can be useful to separate out building tests and running them. Nextest has experimental support for serializing metadata on one machine, and then using that to run tests on another machine.

## Terms

- **Build machine:** The computer that builds tests.
- **Target machine:** The computer that runs tests.

## Use cases

- **Cross-compilation.** The build machine has a different architecture, or runs a different operating system, from the target machine.
- **Test partitioning.** Build once on the build machine, then [partition test execution](partitioning.md) across multiple target machines.
- **Saving execution time on more valuable machines.** For example, build tests on a regular machine, then run them on a machine with a GPU attached to it.

## Requirements

- **The project source must be checked out to the same revision on the target machine.** This might be needed for test fixtures and other assets, and nextest sets the right working directory relative to the workspace root when executing tests.
- **It is your responsibility to transfer over build artifacts.** Use the examples below as a template.

### Non-requirements

- **Cargo does not need to be installed on the target machine.** If `cargo` is unavailable, replace `cargo nextest` with `cargo-nextest nextest` in the following examples.

## New options

Both `cargo nextest list` and `cargo nextest run` accept the following options:

* `--binaries-metadata`: The path to JSON metadata generated by `cargo nextest list --list-type binaries-only --message-format json`.
* `--binaries-dir-remap`: A possible new location for the directory test binaries are present in. Requires `--binaries-metadata`.
* `--cargo-metadata`: The path to JSON metadata generated by `cargo metadata --format-version 1`.
* `--workspace-remap`: A possible new location for the root of the workspace. requires `--cargo-metadata`.

## Example: Simple build/run split

1. Build the tests and save the binaries metadata:
    ```
    cargo nextest list --list-type binaries-only --message-format json \
      > target/binaries-metadata.json
    ```
2. List the tests:
    ```
    cargo nextest list --binaries-metadata target/binaries-metadata.json
    ```
3. Run the tests:
    ```
    cargo nextest run --binaries-metadata target/binaries-metadata.json
    ```

## Example: Cross-compilation

While cross-compiling code, some tests may need to be run on the host platform. (See the note about [Filtering by build platform](running.md#filtering-by-build-platform) for more.)

### On the build machine

1. Save the project's cargo metadata:
    ```
    cargo metadata --format-version=1 --all-features --no-deps \
      > target/cargo-metadata.json
    ```
2. Build the tests and save the binaries metadata:
    ```
    cargo nextest list --target <TARGET> --list-type binaries-only \
      --message-format json > target/binaries-metadata.json
    ```
3. Run host-only tests:
   ```
   cargo nextest run --target <TARGET> --platform-filter host
   ```
4. Archive artifacts: `target/cargo-metadata.json`, `target/binaries-metadata.json`, and test binaries.
    * With [`jq`](https://stedolan.github.io/jq/), this can be done through `cat target/binaries-metadata.json | jq '."rust-binaries" | .[] . "binary-path"`).

### On the target machine

1. Check out the project repository to the same revision as the build machine.
2. Extract the artifacts archived on the build machine.
3. List target-only tests:
    ```
    cargo nextest list --platform-filter target \
        --binaries-metadata <PATH>/binaries-metadata.json \
        --cargo-metadata <PATH>/cargo-metadata.json \
        --workspace-remap <REPO-PATH> \
        --binaries-dir-remap <TESTS-FOLDER-PATH>
    ```
4. Run target-only tests:
    ```
    cargo nextest run --platform-filter target \
        --binaries-metadata <PATH>/binaries-metadata.json \
        --cargo-metadata <PATH>/cargo-metadata.json \
        --workspace-remap <REPO-PATH> \
        --binaries-dir-remap <TESTS-FOLDER-PATH>
    ```