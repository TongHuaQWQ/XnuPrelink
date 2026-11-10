# XnuPrelink

XNU kernelcache prelink and ABI inspection. Fork of **`https://github.com/UniXNU/XnuPrelink.git`**.

## 1. What this is

A Python kit that pairs a **stripped Apple kernelcache** with **our own symbol-bearing rebuilt
kernel** and emits the kernel collection that becomes the boot image's `kernel.macho`. For an
arm64e `MH_FILESET` kernelcache, `emit_prelink.py` selects the fileset emitter: `fileset_match.py`
derives a function/data pairing, `fileset_layout.py` and `fileset_segmented.py` place the native
segments in the space the replaced kernel and the omitted entries free, and the result is a kernel
collection whose root is the native kernel and whose remaining entries are the selected Apple kexts
plus their dependency closure. The ordinary (non-fileset) ARM64 prelink interface remains available.

It is a **prelink-stage, build-time** tool: it sits between the XNU link and the payload. It produces
no runtime component of its own — its output is the kernel image the loader hands to XNU.

Why this fork exists: macOS 26 ships no KDK, so there is no symbol-bearing build of the original
kernel to compare against. Pairing therefore needs hand-reviewed identities per kernelcache build
(`fixtures/fileset-<build>-symbols.json`) and a record of which call site wanted which unnamed kernel
function. Upstream has neither.

## 2. Where it sits

Fork of `https://github.com/UniXNU/XnuPrelink.git` (this checkout: `origin` = upstream,
`mine` = `TongHuaQWQ/XnuPrelink`).

Who pins it:

| Consumer | Pin | Where |
| --- | --- | --- |
| core `TongHuaQWQ/UniXNU` | submodule `kits/XnuPrelink` → `https://github.com/TongHuaQWQ/XnuPrelink.git`; catalog `revision = "a7714579cb9f1243a74abe386dc18a5174da881e"`, `path = "kits/XnuPrelink"`, `build_stage = "prelink"`, `legacy_name = "NativePrelink"` | `UniXNU/.gitmodules`, `UniXNU/components.toml` (`id = "XnuPrelink"`) |
| device `TongHuaQWQ/macOS-ForUke` | `[[source]] name = "XnuPrelink" kind = "kit" local = "kits/XnuPrelink"` — **a board copy inside the device repository**, no git pin; `[kernel.prelink] tool = "kits/XnuPrelink/emit_prelink.py"` | `macOS-ForUke/feeds.toml`, `macOS-ForUke/board.toml` |

> **The board copy is what CI actually runs.** `feeds.py` resolves sources first-wins, and a `git`
> entry cannot override a component the core already carries as a submodule, so the uke board keeps
> a writable copy at `macOS-ForUke/kits/XnuPrelink` (also reachable as the compatibility symlink
> `kits/NativePrelink`). The copy is **ahead of this repository**: its
> `fixtures/fileset-25F84-symbols.json` carries 197 reviewed identities against 60 here, and only the
> copy has `fixtures/reference-names-25F84-kernel.release.t8142.json.gz` and
> `test_reference_names.py`. The two READMEs are kept consistent; the code is not yet. Treat the
> board copy as the branch of record until those commits are pushed back here.

What it links into: the kernel collection (the payload's `kernel.macho`), not a static archive. The
kernel archives themselves are built by the core's `scripts/build-*.py`; this kit only consumes the
linked native kernel.

## 3. Dependencies

| Input | Comes from | Without it |
| --- | --- | --- |
| Python 3.11+ (standard library only — `struct`, `json`, `hashlib`, `hmac`, `plistlib`, `xml.etree`, `array`, `bisect`) | Host | Nothing runs. There is no third-party dependency and no compiler step. |
| A decompressed `MH_FILESET` kernelcache | The target IPSW, via `unixnu ipsw fetch` (IM4P payload → LZFSE decode → the single `0x0100000c` slice); pinned by the OS profile's `kernelcache_sha256` | No original collection to pair against: the emitter has no input. When a `--fileset-map` is given, **both** input hashes (native kernel and kernelcache) must match before the map is reused. |
| The linked native kernel **with symbols** | The board's kernel build (`unixnu build kernel` stages it; `kernel.json` records its path) | Nothing to emit: the collection root is the native kernel. `unixnu --skip-prelink` stages it unchanged as a `native-only` image instead of prelinking. |
| A symbol-bearing comparison kernel | `scripts/build-fileset-release-reference.py --tree . [--lto]`, run as the last entry of the board archive list | Fewer names transfer: matches fall back to code shape, data operands and identified vtables. On macOS 26 this is the only alternative to the shipped naming table. |
| The shipped naming table (`reference-names.json[.gz]`) | Generated once from a symbol-bearing kernel (`tools/build_reference_names.py` in the UniXNU measurement directory) and committed as a fixture | Same as above; the map records the table hash and is refused if a different table is supplied. |
| Reviewed anchors in `fixtures/` | **Hand-made** from identification evidence (function hashes, `__PRELINK_INFO`, sequence alignment, `vdesc_name`, vtable slots). Apple publishes no KDK for macOS 26, so this is not obtainable from any public source | Ambiguous functions stay unbound; the pairing reports them as missing. |
| The original kernelcache's own `com.apple.kernel` symbol table | Inside the kernelcache (`declared_names()`) | `--root`'s dependency closure cannot resolve *data* names, and `unresolved closure references` stays non-zero. |

Successful emission and the structural checks do **not** establish ABI or boot compatibility:
pairings are inferred from stripped code.

## 4. Build and use

There is nothing to compile. Host regressions:

```sh
python3 -m unittest discover -p 'test_*.py'      # 59 tests, ~0.03 s
```

The emitter, as the build driver calls it:

```sh
python3 emit_prelink.py --kernel <native kernel> --prelinked <decompressed kernelcache> \
    --root <com.apple.bundle.id> [--root ...] \
    --slide <int> --output <kernel.macho> \
    [--fileset-map FILE] [--fileset-reference FILE] [--allow-missing] \
    [--early-security-init <BUNDLE>=<ORIGINAL_VA>]
```

`--slide` is required by the CLI but does not apply to filesets (rebuilt chains use virtual offsets).
Maps carry a matcher version as well as the input hashes; the integrated CLI regenerates outdated
caches, and both emission entry points reject maps produced by outdated matching code. Missing
mappings stop output and are listed in the JSON report beside the requested image; `--allow-missing`
explicitly permits an experimental image with unresolved references.

The standalone matcher (its `--reference-kernel` option is the emitter's `--fileset-reference`):

```sh
python3 fileset_match.py <decompressed MH_FILESET kernelcache> <native arm64e kernel with symbols> \
    [--entry com.apple.kernel] [--out pairing.json] [--reference-kernel FILE] [--max-rounds 40]
```

The lower-level `fileset_emit.py` accepts `--map FILE`, `--output FILE`, `--report FILE` and repeated
`--root BUNDLE_ID` arguments.

## 5. See also

- [Port a device](https://unixnu.com/zh/porting/steps) — step 2 (choose the shared support, including
  the prelink ABI work) and step 7 (build and boot);
- [Prelink concept](https://unixnu.com/zh/concepts/prelink) and [payload concept](https://unixnu.com/zh/concepts/payload);
- [XnuPrelink component page](https://unixnu.com/zh/drivers/xnu-prelink) and the
  [component catalog](https://unixnu.com/zh/reference/components);
- [Drivers](https://unixnu.com/zh/porting/drivers), [log capture](https://unixnu.com/zh/porting/log-capture);
- in the core repository: `docs/drivers/xnu-prelink.md`, `docs/concepts/prelink.md`,
  `components.toml`; in the device repository: `porting.toml`, `docs/STATE.md`, `board.toml`
  `[kernel.prelink]`.

## Behaviour notes

`emit_prelink.py` selects the fileset path for an `MH_FILESET` input. It generates a function/data
pairing with `fileset_match.py`, or reads one from `--fileset-map FILE`. Both input hashes must match
before an existing map can be reused.

The fileset emitter places the collection root at the native kernel's link address, marks the copied
kernel as a fileset member, rebases metadata and keeps only the selected roots and their dependencies
in the fileset index. Rebuilt chains use virtual offsets, including across native BSS gaps.

`fileset_layout.py` places native segments in space freed by the replaced kernel and omitted entries.
Code follows retained kext code; constants and data precede their kext ranges as required by XNU's
`arm_vm_init`. `fileset_segmented.py` applies native split-segment v2 records, rebuilds root fixups
and function starts, and recomputes corecrypto's HMAC over relocated code. Segment collisions and
unsupported relocation kinds stop emission. The native input executable is unchanged.

Reviewed symbol anchors in `fixtures/` include function hashes and identification evidence. Other
matches use code, data operands and identified vtables. A `.cold` error path is not substituted for
its parent function. Link order alone does not bind ambiguous functions. The comparison recognizes
complete compiler-inserted PAC failure checks and preserves branch geometry when comparing bodies;
native code keeps those checks. `--fileset-reference FILE` accepts a symbol-bearing comparison kernel,
such as a RELEASE build with LTO: unique code matches may supply an unambiguous symbol name in the
actual native kernel, while reference addresses are never emitted and compiler-generated names and
graph-only guesses do not transfer.

In the UniXNU build driver, `unixnu prelink --skip-prelink` stages an existing build. This path copies
the native bytes unchanged and does not fetch a kernelcache or add its kexts. Its adjacent JSON record
marks the output as `native-only`; payload arguments and loader support are separate.

Build and interface details: [documentation](https://unixnu.com/drivers/xnu-prelink). Component
sources and dependencies: [catalog](https://unixnu.com/reference/components).
