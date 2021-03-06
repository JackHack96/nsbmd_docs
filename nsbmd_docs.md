# Nintendo DS NSBMD Model Format Docs

This is documentation for the binary format of the `NSB__` files (`NSBMD`, `NSBTX`,
etc.) used for 3D models, animations, etc. in Nintendo DS games.

You will need the GBATEK docs on the NDS GPU handy while reading:
<https://problemkaputt.de/gbatek.htm#ds3dvideo>.

**DOCUMENTATION IS BASED ON REVERSE ENGINEERING. ASSUME IT IS SPECULATIVE,
INCOMPLETE, AND CONTAINS ERRORS**.

Documented formats:

- NSBMD (models, textures, palettes)
- NSBTX (textures, palettes)
- NSBCA (skeletal animations)
- NSBTP (pattern animations)
- NSBTA (material animations)

Undocumented formats:

- NSBMA
- NSBVA

Previous documentation:

- kiwi.ds NSBMD docs
  
  <http://sites.google.com/site/kiwids/nsbmd.html>
- lowlines Nitro docs

  <https://web.archive.org/web/20180326163503/http://llref.emutalk.net/docs/>
- Gericom's EveryFileExplorer (Source Code)

  <https://github.com/Gericom/EveryFileExplorer/blob/master/NDS/NitroSystem/G3D/>

This document is based on practical experience implementing a viewer:
<https://github.com/scurest/apicula>.

## Terminology

I have heard the etymology NSBMD = "New Super Mario Bros. Model Data".

All the names in this document (eg. Mesh, BoneMatrix, etc.) have been invented
for expository purposes.

## Conventions

All data is little-endian.

`u8`, `u16`, and `u32` are unsigned 8, 16, and 32-bit unsigned integers, respectively.

`num(X.Y.Z)` is a fixed-point number. The next `(X+Y+Z)` bits should be read
and interpreted as an unsigned integer `(if X is 0)` or a signed twos-complement
integer `(if X is 1)`. The denoted number is this integer times 2^(-Z).

Example:
> `num(1.19.12)` is the number type used by the DS GPU. Note it is
> `u32`-sized. In it, `0x1000` represents the number `1`.

Arrays are written `type[length]`, ex. `u8[3]` is an array of three `u8s`.

Arrays of unknown length are written `type[]`. You can index into these if you
have an index, but you can't iterate over them.

Offsets usually have names ending with `_off`. Offset are always in bytes. I'll
always say what an offset is relative to. Add the offset to the location of
whatever its relative to to find the actual data.

Bit-fields are written like this

```
u8 {
    low_nibble:  bits(0,4)
    high_nibble: bits(4,8)
}
```

Matrix conventions are as in mathematics: Vectors are column-vectors.
Application of a matrix `M` to a vector `v` is `Mv`. Matrix multiplication `AB` is "B
followed by A". (Note: this is the opposite convention of the one in the
GBATEK docs.)

## Common Idioms

These elements occur in multiple places.

A Name is a human-readable, null-padded, 16-byte ASCII string.

```
Name {
    name:    u8[16]
}
```

A `NameList(T)` (`T` is some type) is a list of `Ts`, each `T` having a Name. Usually `T`
is going to be an offset to where the actual data for that element is located.
Names appear to always be unique within a `NameList`.

```
NameList(T) {
    dummy:    u8
    count:    u8   (number of elements)
    size:     u16  (total size of this NameList in bytes)

    Unknown {
        UnknownHeader {
            subheader_size:  u16  (total size of this UnknownHeader in bytes?)
            unknown_size:    u16  (total size of this Unknown in bytes?)
            unknown:         u32
        }
        unknown:    u32[count]
    }

    element_size:       u16      (size of T in bytes)
    data_section_size:  u16

    data:     T[count]
    names:    Name[count]
}
```

## Containers

A Container is the top-level object in the binary format of a `NSB__` file. A
Container holds subfiles. Subfiles, in turn, hold individual models, animations,
etc.

Containers and subfiles start with a four-byte stamp (magic number) that
identifies what kind of data it stores.

```
Container {
    Header {
        stamp:           u8[4]   (depends on kind of Container, see below)
        bom:             u16     (byte order mark, =0xfeff)
        version:         u16     (version 2?)
        filesize:        u32
        header_size:     u16     (size of this Header?; always 16)
        num_subfiles:    u16
    }

    subfile_offs:    u32[num_subfiles]
    (each u32 points to a subfile relative to the Container)
}
```

The different kinds of Containers follow:

- An `NSBMD` contains `MDLs` (usually one) and `TEXs` (usually zero or one).
- An `NSBTX` contains `TEXs`.
- An `NSBCA` contains `JNTs`.
- An `NSBTP` contains `PATs`.
- An `NSBTA` contains `SRTs`.

```
NSBMD {
    Container  (expected stamp = "BMD0")
}
NSBTX {
    Container  (expected stamp = "BTX0")
}
NSBCA {
    Container  (expected stamp = "BCA0")
}
NSBTP {
    Container  (expected stamp = "BTP0")
}
NSBTA {
    Container  (expected stamp = "BTA0")
}
```

## Models

An `MDL` is a subfile containing Models.

```
MDL {
    stamp:        u8[4]  (= "MDL0")
    filesize:     u32    (total filesize of this MDL)

    models:   NameList(u32)
    (each u32 gives the offset to a Model relative to this MDL)
}
```

A `Model` is 3D model. The process of drawing a `Model` consists of executing a list
of `RenderCommands`, which calculate skinning matrices, set material properties,
and draw the individual pieces of the `Model` (the `Meshes`).

```
Model {
    filesize:          u32    (size of Model+all its data?)

    (all offsets are relative to this Model)
    render_cmds_off:   u32    (points to RenderCommandList)
    materials_off:     u32    (points to MaterialList)
    meshes_off:        u32    (points to MeshList)
    inv_binds_off:     u32    (points to InvBindMatrices)

    unknown:           u8[3]
    num_bone_matrices: u8
    num_materials:     u8
    num_meshes:        u8
    unknown:           u8[2]

    (used by certain rendering commands)
    up_scale:          num(1.19.12)
    down_scale:        num(1.19.12)

    num_verts:         u16
    num_polys:         u16    (num_tris + num_quads?)
    num_tris:          u16
    num_quads:         u16

    BoundingBox        (12 bytes total)

    unknown:           u8[8]

    BoneList
}

BoundingBox gives the model's bounding box?
TODO: verify.

BoundingBox {
    x_min:    num(1.3.12)
    y_min:    num(1.3.12)
    z_min:    num(1.3.12)

    x_max:    num(1.3.12)
    y_max:    num(1.3.12)
    z_max:    num(1.3.12)
}
```

---

A `MeshList` stores `Meshes`.

```
MeshList {
    NameList(u32)    (each u32 is the offset of a Mesh relative to this MeshList)
}
```

A `Mesh` contains actual vertex data in the form of a blob of NDS GPU commands. To
draw a `Mesh`, you just submit the blob of commands to the GPU.

```
Mesh {
    dummy:     u16
    size:      u16   (=16, possibly the size of Mesh?)
    unknown:   u32
    cmds_off:  u32   (relative to this Mesh)
    cmds_len:  u32
}
```

`cmds_off` points to a `u8[cmds_len]` blob containing the GPU commands.

A blob of GPU commands is stored as a sequence of packets. Each packet encodes
four GPU commands as a sequence of `u32s`. A GPU command is an 8-bit opcode and
some number of `u32` parameters. The first `u32` in a packet gives the four opcodes
of the commands, followed by the parameters of the first command, then the
parameters of the second, etc.

See the GBATEK docs for more info about the binary format and semantics of GPU
commands.

Only certain commands appear in a `Mesh`. Here is a list:

```
NOP (0x0)         MTX_RESTORE (0x14)   MTX_SCALE (0x1b)    BEGIN_VTXS (0x40)
END_VTXS (0x41)   VTX_16 (0x23)        VTX_10 (0x24)       VTX_XY (0x25)
VTX_XZ (0x26)     VTX_YZ (0x27)        VTX_DIFF (0x28)     TEXCOORD (0x22)
COLOR (0x20)      NORMAL (0x21)
```

---

The `RenderCommandList` is the script that you run to draw the `Model`. Each render
command consists of a `u8` opcode and some number of `u8` parameters.

```
RenderCommandList {
    loop {
        RenderCommand {
            opcode:      u8
            parameters:  u8[n]  (n depends on the opcode; see below)
        }
        if opcode == 1 {
            break;
        }
    }
}
```

The low five bits of the opcode determine the operation to perform. The three
high bits modify the behavior of the operation.

Known render commands:
```
Nop (0x00, 0x40, 0x80)
  0 parameters
  Does nothing? Difference between opcodes is unknown.

End (0x01)
  0 parameters
  Marks the end of the RenderCommandList.

Unknown (0x02)
  2 parameters

Load Matrix from Stack (0x03)
  1 parameter
    * stack slot to load from
  Loads a stack matrix into the current matrix.

    cur_matrix = matrix_stack[next_parameter()]

Bind Material (0x4, 0x24, 0x44)
  1 parameter
    * index of the material to bind
  Bind a material for subsequent draw commands. Difference between opcodes is
  unknown.

Draw Mesh (0x05)
  1 parameter
    * index of the Mesh to draw
  Draws a Mesh.

Multiply Current Matrix with Bone Matrix (0x06, 0x26, 0x46, 0x66)
  3 parameters if opcode == 0x06
  4 parameters if opcode == 0x26 or 0x46
  5 parameters if opcode == 0x66
    * bone_idx: index of the BoneMatrix to multiply with
    * parent_idx: apparently, the index of the parent of the bone from bone_idx
    * unknown
    * (if opcode & 0x40) stack slot to load from beforehand
    * (if opcode & 0x20) stack slot to store to afterward
  Multiplies the current matrix by a BoneMatrix. This is used to build the
  local-to-world matrices out of the BoneMatrices. If the 0x40 bit of the opcode
  is set, load a matrix from the stack beforehand. If the 0x20 bit is set, store
  the matrix to the stack afterward.

    bone_idx = next_parameter()
    parent_idx = next_parameter()
    unknown = next_parameter()

    if opcode & 0x40 {
        cur_matrix = matrix_stack[next_parameter()]
    }
    cur_matrix *= bone_matrices[bone_idx]
    if opcode & 0x20 {
        matrix_stack[next_parameter()] = cur_matrix
    }

  NOTE: If bone_idx is the index of bone B, then parent_idx will be the index of
  the parent of B. I don't think it's used at all at runtime.

Unknown (0x07, 0x47)
  1 parameter (2 for 0x47?)

Unknown (0x08)
  1 parameter

Calculate Skinning Equation (0x09)
  Parameters (variable number):
    * store_index: the stack index at which to store the calculated matrix
    * number of terms (determines how many parameters will follow)
    loop number of term times {
        * stack_index: index into the matrix stack to use as the local-to-world matrix
        * inv_bind_idx: index into InvBindMatrices to use as the inverse bind matrix
        * weight: stored normalized; divide by 256 to get the actual value
    }
  Calculates a matrix with the skinning equation

    ∑_B (weight for B) * (local-to-world for B) * (inverse bind for B)

  This is the matrix applied to a vertex influenced by multiple bones. The
  inverse bind matrices bring the vertex into the local space of each bone and
  the local-to-world transforms send it to its world space position.

  By contrast, if a vertex is only influenced by a single bone, then its
  position will just be stored (in the Mesh) in the space of that bone, so
  there's no need for an InvBindMatrix to bring it into the correct space. So
  this command is only used when there are vertices influenced by multiple
  bones.

    cur_matrix = 0
    store_index = next_parameter()
    num_terms = next_parameter()
    loop num_terms times {
        term = matrix_stack[next_parameter]
        term *= inv_bind_matrices[next_parameter()].matrix
        term *= next_parameter() / 256
        cur_matrix += term
    }
    matrix_stack[store_index] = cur_matrix

Scale Up (0x0b)
Scale Down (0x2b)
  0 parameters
  Scales the current matrix by the value Model.up_scale (resp.
  Model.down_scale).

    cur_matrix *= Model.up_scale (or Model.down_scale if opcode == 0x2b)

Unknown (0x0c)
  2 parameters

Unknown (0x0d)
  2 parameters
```

---

A `MaterialList` contains `Materials`, and what texture/palette Name they should be
paired with.

```
MaterialList {
    texture_pairings_off:   u16   (relative to this MaterialList)
    palette_pairings_off:   u16   (relative to this MaterialList)

    NameList(u32)       (each u32 is the offset of a Material relative to this MaterialList)
}
```

A `Material` is a bunch of GPU state (eg. colors, whether backface culling is
enabled, etc.) to be set when the `Material` is bound. It also determines the
texture/palette to use, though that isn't stored in this `Material` object itself.

```
Material {
    dummy:             u16
    size:              u16    (size of this Material in bytes)

    (the following u32s give the parameters to GPU commands to be submitted
     when this Material is bound)
    dif_amb:           u32    (parameter to DIF_AMB (opcode 0x30))
    spe_emi:           u32    (parameter to SPE_EMI (opcode 0x31))
    polygon_attr:      u32    (parameter to POLYGON_ATTR (opcode 0x29);)
    unknown:           u32    (possibly parameter to SHININESS (opcode 0x34)??)
    teximage_params:   u32    (parameter for TEXIMAGE_PARAMS (opcode 0x2a); see below)

    unknown:           u32
    unknown:           u32

    texture_width:     u16
    texture_height:    u16

    (TODO: the remaining fields are unknown but should comprise at least the
     texcoord transform matrix, if used)
}
```

`teximage_params` is the `u32` parameter to the GPU command `TEXIMAGE_PARAMS` (opcode
`0x2a`). See the GBATEK documentation for details. Only some of its fields are
stored here; the others are zeroed out. They are stored in the teximage_params
in the Texture object. These two teximage_param `u32s` are or-ed together to give
the final argument to `TEXIMAGE_PARAMS`. The fields stored here are:

```
u32 {
    repeat_s:     bits(16,17)
    repeat_t:     bits(17,18)
    mirror_s:     bits(18,19)
    mirror_t:     bits(19,20)
    texcoord_transform_mode:  bits(30,32)
}
```

`TexturePairings` and `PalettePairings` pair `Materials` with the `Names` of the
textures/palettes they should use. The precise mechanism by which a `Name` is
resolved to an actual texture or palette is unknown but it appears to be at
least partially controllable from game code. In the simplest case, there will be
a `TEX` in the same `NSBMD` as this Model, and you can look for a `Texture`/`Palette`
with the given Name there.

```
TexturePairingList {
    NameList(MaterialIdxList)
    (each Name gives a texture name that applies to all the Materials in the MaterialIdxList)
}

PalettePairingList {
    NameList(MaterialIdxList)
    (same as TexturePairingList, but for palettes obviously)
}

MaterialIdxList {
    offset:    u16    (points to a u8[count]; relative to this MaterialIdxList)
                      (each u8 is the index of a Material)
    count:     u8
    dummy:     u8
}
```

This somewhat unusual way of associating textures/palette Names with `Materials`
(as opposed to simply having a texture_name/palette_name field in the `Material`)
saves space when many `Materials` share texture/palette `Names` or have no
texture/palette. (Is this its only goal?)

---

A `BoneList` stores `BoneMatrices`.

```
BoneList {
    NameList(u32)    (each u32 points to a BoneMatrix; relative to this BoneList)
}
```

A `BoneMatrix` stores the local-to-parent transform of some bone. A `BoneMatrix` is
a TRS transform (that is, it consists of a scaling, followed by a rotation,
followed by a translation). It is determined by seven TRS properties
```
translation X       rotation       scale X
translation Y                      scale Y
translation Z                      scale Z
```

Translation and scale components are real numbers. The rotation is a 3x3 matrix.
The binary coding for rotation matrices tends to favor true rotations (ie.
orthogonal matrices), although it is possible to encode a non-rotation matrix as
the "rotation" of a TRS transform.

When a `Model` is animated by an `Animation`, the only thing that changes are its
`BoneMatrices`.

**NOTE**: a `Model` doesn't contain any actual bones or skeleton information (but see
the `parent_idx` parameter to render command `0x06`); that has all been compiled
down to an imperative list of rendering commands that build up all the necessary
skinning matrices directly. The `BoneMatrices` just store the data from the bones
that are needed by these rendering commands.

```
BoneMatrix {
    u16 {
        t:         bits(0,1)    (controls if there's a translation)
        rm:        bits(1,2)    (controls if there's a rotation given by matrix entries)
        s:         bits(2,3)    (controls if there's a scale)
        rp:        bits(3,4)    (controls if there's a rotation given by a PivotMatrix)

        (these are used for the rotation matrix if rp == 1)
        form:      bits(4,8)
        neg_one:   bits(8,9)
        neg_c:     bits(9,10)
        neg_d:     bits(10,11)

        ignored:   bits(11,16)
    }

    (used for rotation matrix; it's here for alignment reasons)
    m0:      num(1.3.12)

    (translation; default is (0 0 0))
    if t == 0 {
        Translation {
            x:   num(1.19.12)
            y:   num(1.19.12)
            z:   num(1.19.12)
        }
    }

    (rotation; default is identity matrix)
    if rp == 1 {
        a:        num(1.3.12)
        b:        num(1.3.12)
        (these determine a PivotMatrix; see below)
    } else if rm == 0 {
        ms:       num(1.3.12)[8]
        (the rotation matrix is given by
            [  m0   ms[2] ms[5] ]
            [ ms[0] ms[3] ms[6] ]
            [ ms[1] ms[4] ms[7] ]
        )
    }

    (scale; default is (1 1 1))
    if s == 0 {
        Scale {
            x:   num(1.19.12)
            y:   num(1.19.12)
            z:   num(1.19.12)
        }
    }
}
```

If `rp == 1`, the six values

```
form     neg_one     neg_c     neg_d      a       b
```

determine the rotation matrix as though it were given by a `PivotMatrix`; see the
definition of `PivotMatrix` in the animation section.

---

`InvBindMatrices` are 4x4 matrices used in computing the skinning matrix for
render command `0x09`. If a `Model` doesn't use this command, it doesn't need to
have any, but many `Models` have them anyway.

```
InvBindMatrices {
    InvBindMatrix[]
}

InvBindMatrix {
    matrix:   num(1.19.12)[12]    (3x4 matrix; column-major order)
    unknown:  num(1.19.12)[9]     (often seems to be the linear part of matrix; used for normals maybe?)
}
```

Only the upper 3x4 block of the 4x4 inverse bind matrix is stored; the final row
is always `(0 0 0 1)`. (NOTE: the NDS GPU can do math on 3x4 matrices directly.)

## Animations

A JNT is a subfile containing `Animations`.

```
JNT {
    stamp:        u8[4]    (= "JNT0")
    filesize:     u32      (total filesize of this JNT)

    animations:   NameList(u32)
    (each u32 gives the offset to an Animation relative to this JNT)
}
```

An `Animation` is a skeletal animation of a `Model`. It changes the values of the
BoneMatrices in a `Model` over time. All animations are frame-based, so "time"
always means "frame number".

An `Animation` contains a collection of `Tracks`. A `Track` targets one of the
`BoneMatrices` in a `Model` and tells you what its value should be at each time.

```
Animation {
    unknown:       u8[4]     (="J\0AC", is this a stamp?)
    num_frames:    u16
    num_tracks:    u16
    unknown:       u32

    pivot_data_off:      u32  (points to PivotMatrices; relative to this Animation)
    basis_matrices_off:  u32  (points to BasisMatrices; relative to this Animation)

    track_offs:  u16[num_tracks]
    (each offset points to an AnimationTrack; relative to this Animation)
}
```

`PivotMatrices` and `BasisMatrices` store rotation matrices that will be used in the
curve data below

```
PivotMatrices {
    PivotMatrix[]
}
BasisMatrices {
    BasisMatrix[]
}
```

A `Track` consists of channels. A channel is a connection between one of the seven
TRS properties and a `Curve`, saying what the value of that TRS property should be
over time.

**OPEN QUESTION**: if a track doesn't have a channel for a particular TRS property,
what should the value of that property be? (eg. should it retain the value it
has in the Model?)

```
Track {
    (this u16 determine which channels are present, and, if they are present,
     whether their Curve is constant or sampled (see below))
    u16 {
        no_channels: bits(0,1)
        no_translation_channels: bits(1,3)
            translation X is constant: bits(3,4)
            translation Y is constant: bits(4,5)
            translation Z is constant: bits(5,6)
        no_rotation_channel: bits(6,8)
            rotation is constant: bits(8,9)
        no_scale_channels: bits(9,11)
            scale X is constant: bits(11,12)
            scale Y is constant: bits(12,13)
            scale Z is constant: bits(13,14)
    }

    dummy:          u8
    target_index:   u8  (the index of the BoneMatrix this track targets?)

    if no_channels {
        return;
    }

    if no_translation_channels == 0 {
        Curve     (translation X)
        Curve     (translation Y)
        Curve     (translation Z)
    }
    if no_rotation_channel == 0 {
        Curve     (rotation)
    }
    if no_scale_channels == 0 {
        Curve     (scale X)
        Curve     (scale Y)
        Curve     (scale Z)
    }
}
```

A `Curve` is a function mapping time to values (either a real number for
translation/scale components, or a 3x3 matrix for rotations). It can either be
constant, in which case it is defined by a single value

```
  ^
  |
  |_____________
  |
  |
  '------------->
    time
```

or sampled, in which case it is defined by a set of discrete (frame number,
sample value) pairs.

```
  ^        .
  |  .     |
  |  |  .  |
  |  |  |  |  .
  |  |  |  |  |
  '--+--+--+--+->
    time
```

**OPEN QUESTION**: how should a sampled curve be evaluated at a frame between two
sample times?

```
Curve {
    if curve is constant {
        (the constant value of the curve follows; depends on the type of
         channel)

        if translation channel {
            constant_value:   num(1.19.12)
        }
        if rotation channel {
            constant_value:   RotMatrixIdx
            ignored:          u16    (padding for alignment, probably)
        }
        if scale channel {
            constant_value:   num(1.19.12)
            unknown:          num(1.19.12)
        }
    } else {
        (a sampled curve; see explanation below)
        u32 {
            start_frame:  bits(0,16)
            end_frame:    bits(16,28)
            width:        bits(28,30)
            log_rate:     bits(30,32)
        }

        samples_off:      u32    (relative to the containing Animation)
    }
}
```

A sampled curve is always sampled at a fixed rate between two endpoints: one
sample is stored at each of the frames

```
start_frame
start_frame + rate
start_frame + 2*rate
...
end_frame - 2*rate
end_frame - rate
```

The rate is `2^(log_rate)`. Since the `log_rate` field is 2-bits, the possible rates
are 1, 2, 4, and 8. I have never seen 8.

The total number of samples is therefore
```
num_samples = (end_frame - start_frame) / rate
```

**ASSUMPTION**: start_frame and end_frame are divisible by the rate.

`samples_off` points to the array of `num_samples` sample values. The format of a
sample value depends on the type of channel and the width field

```
if translation channel {
    if width == 0 {
        sample:   num(1.19.12)
    } else {
        sample:   num(1.3.12)
    }
}
if rotation channel {
    sample:   RotMatrixIdx
}
if scale channel {
    if width == 0 {
        sample:   num(1.19.12)
        unknown:  num(1.19.12)
    } else {
        sample:   num(1.3.12)
        unknown:  num(1.3.12)
    }
}
```

A `RotMatrixIdx` points to a rotation matrix stored in the `PivotMatrices` or
`BasisMatrices` arrays for this `Animation`. The highest bit tells you which array
it's in, and the low bits give the index into that array.

```
RotMatrixIdx {
    u16 {
        index: bits(0,15)
        is_pivot: bits(15,16)
    }
}
```

If `is_pivot` is set, use `PivotMatrices[index]`; otherwise, use
`BasisMatrices[index]`.

---

A `PivotMatrix` encodes a rotation matrix in 3 `u16s`. It is good for representing
rotations where the axis of rotation is the X, Y, or Z axis.

```
PivotMatrix {
    u16 {
        form:     bits(0,4)
        neg_one:  bits(4,5)
        neg_c:    bits(5,6)
        neg_d:    bits(6,7)
        ignored:  bits(7,16)
    }
    a:        num(1.3.12)
    b:        num(1.3.12)
}
```

- Let i = +1 if neg_one is unset; -1 if it is set
- Let c = +a if neg_c is unset; -a if it is set
- Let d = +b if neg_d is unset; -b if it is set

The final matrix then depends on form as
```
If form=0    If form=1    If form=2
[ i     ]    [   a c ]    [   a c ]
[   a c ]    [ i     ]    [   b d ]
[   b d ]    [   b d ]    [ i     ]

If form=3    If form=4    If form=5
[   i   ]    [ a   c ]    [ a   c ]
[ a   c ]    [   i   ]    [ b   d ]
[ b   d ]    [ b   d ]    [   i   ]

If form=6    If form=7    If form=8
[     i ]    [ a c   ]    [ a c   ]
[ a c   ]    [     i ]    [ b d   ]
[ b d   ]    [ b d   ]    [     i ]
```

---

A `BasisMatrix` encodes a rotation matrix in 5 `u16s`. It encodes an arbitrary
rotation by storing the 6 entries in the first two columns of the 3x3 matrix;
the third column is then uniquely determined (by the cross-product).

```
BasisMatrix {
    xs:    u16[5]
}
```

The precise computation for the matrix is extremely odd. There is probably some
way to rewrite this function that makes it make sense. Credit for figuring this
out goes to MKDS Course Modifier.

- Let `ys = [xs[4], xs[0], xs[1], xs[2], xs[3]]`.
- Let `zs = [0, 0, 0, 0, 0, 0]`.

```
for i=0,1,2,3,4 {
    zs[i] = ys[i].bits(3,16)
    zs[5] <<= 3
    zs[5] |= ys[i].bits(0,3)
}
```

The elements of `zs` are 13-bit numbers. Interpret them as num(1.0.12)s.

```
        [ zs[1] ]
Let A = [ zs[2] ].
        [ zs[3] ]

        [ zs[4] ]
Let B = [ zs[0] ].
        [ zs[5] ]

Let C = AxB (the cross-product of A and B).
```

Then A, B, and C are the columns of the final matrix

```
  [ | | | ]
  [ A B C ]
  [ | | | ]
```

## Pattern Animations

A PAT is a subfile containing PatternAnimations.

```
PAT {
    stamp:        u8[4]    (= "PAT0")
    filesize:     u32      (total filesize of this PAT)

    pattern_animations:   NameList(u32)
    (each u32 gives the offset to a PatternAnimation relative to this PAT)
}
```

A `PatternAnimation` is an animation that varies the texture/palette the Materials
in a `Model` use over time.

```
PatternAnimation {
    unknown:            u8[4]
    num_frames:         u16
    num_texture_names:  u8
    num_palette_names:  u8
    texture_names_off:  u16    (relative to this PatternAnimation)
    palette_names_off:  u16    (relative to this PatternAnimation)

    tracks:             NameList(Track)
}
```

`texture_names_off` points to a list of texture names that will be used by the
`Tracks`; similarly for `pattern_names_off`.

```
TextureNames {
    Name[num_texture_names]
}

PaletteNames {
    Name[num_palette_names]
}
```

Each `Track` targets a `Material` with the same Name as the `Track`, and tells you
when its texture/palette should change.

```
Track {
    num_keyframes:   u32
    unknown:         u16

    offset:          u16
    (points to a Keyframe[num_keyframes]; relative to the containing PatternAnimation)
}
```

Each `Keyframe` says that the texture/palette `Names` should change to the given
values at the given frame. The values hold until they are changed at the next
`Keyframe`. A `Track`'s array of `Keyframes` is sorted by frame.

```
Keyframe {
    frame:         u16
    texture_idx:   u8    (index into TextureNames to use as texture Name)
    palette_idx:   u8    (index into PaletteNames to use as palette Name)
}
```

## Material Animations

**(Warning: This section is highly incomplete!)**

An SRT is a subfile containing MaterialAnimations.

```
SRT {
    stamp:        u8[4]    (= "SRT0")
    filesize:     u32      (total filesize of this SRT)

    material_animations:   NameList(u32)
    (each u32 gives the offset to a MaterialAnimation relative to this SRT)
}
```

A `MaterialAnimation` is an animation that varies `Material` parameters in a `Model`.
For example, it can vary UV translation to do texture scrolling effects.

```
MaterialAnimation {
    unknown:          u8[4]  ("M\0AT"?)
    num_frames:       u16
    unknown:          u16

    tracks:           NameList(Track)
}
```

Each `Track` targets a `Material` with the same `Name` as the `Track`, and animates its
parameters. A `Track` consists of 5 Channels.

```
Track {
    unknown_channels:       Channel[3]

    (targets the U-translation for texture UVs)
    u_translation_channel:  Channel

    (targets the V-translation for texture UVs)
    v_translation_channel:  Channel
}
```

```
Channel {
    num_frames:   u16
    dummy?:       u8      (always 0?)
    flags?:       u8      (typically has one or two bits set, so possibly flags)

    if flags == 16 {
        offset:   u32
    } else {
        unknown:  u8[4]
    }
}
```

For `channels[3]` and `channels[4]`, if `flags == 16`, then offset points to a
`num(1.10.5)[num_frames]` array containing the values of the UV offset at each
frame.

Other cases are unknown.

## Textures & Palettes

A texture is a 2D array of texels. There are seven different texture formats on
the DS's GPU, numbered 1-7. Texture format 7 encodes actual colors in its
texels, but all the others must be used with a palette that determines the color
each texel value should have. For details of texturing on the DS and how to
decode textures, see the GBATEK documentation.

A `TEX` is a subfile containing textures and palettes. However, unlike the other
subfiles, it is not divided into independently stored objects. Instead, it holds
blocks of data that are shared by all the textures/palettes it contains. AIUI
game code would transfer the blocks into VRAM at load time and then be able to
use any of textures/palettes in the `TEX`.

```
TEX {
    stamp:                 u8[4]     (= "TEX0")
    unknown:               u32
    unknown:               u32
    block1_len_shr_3:      u16
    textures_off:          u16       (points to TextureList; relative to this TEX)
    unknown:               u32
    block1_off:            u32       (points to Block1; relative to this TEX)
    block2_len_shr_3:      u16
    unknown:               u16
    unknown:               u32
    block2_off:            u32       (points to Block2; relative to this TEX)
    block3_off:            u32       (points to Block3; relative to this TEX)
    unknown:               u32
    block4_len_shr_3:      u16       (points to Block4; relative to this TEX)
    unknown:               u16
    palettes_off:          u32       (points to PaletteList; relative to this TEX)
    block4_off:            u32
}
```

The lengths of data blocks are stored shifted right by 3 (that's what `shr_3`
means); shift them left by 3 to get the actual length.

`Block1` stores texture data for all texture formats except 5.

```
Block1 {
    u8[block1_len_shr_3 << 3]
}
```

`Block2` and `Block3` store data for textures with format 5. These are
block-compressed textures: each 4x4 block of texels is compressed into one `u32`
and one `u16`. The data for a compressed texture is therefore two parallel arrays,
one of `u32s` and one of `u16s`. The former is stored in `Block2`, the latter in
`Block3`. For this reason, `Block3` is always half the length of `Block2` (in bytes)
and texture data at offset X into `Block2` is at offset X/2 into `Block3`.

```
Block2 {
    u8[block2_len_shr_3 << 3]
}

Block3 {
    u8[block2_len_shr_3 << 2]
}
```

`Block4` stores palette data.

```
Block4 {
    u8[block4_len_shr_3 << 3]
}
```

A `TextureList` contains `Textures`.

```
TextureList {
    NameList(Texture)
}

Texture {
    teximage_params:    u32
    unknown:            u32
}
```

`teximage_params` is the `u32` parameter to the GPU command `TEXIMAGE_PARAMS` (opcode
`0x2a`). See the GBATEK documentation for details. Only some of its fields are
stored here; the others are zeroed out. They are stored in the teximage_params
in a Material that uses this Texture. These two teximage_param `u32s` are or-ed
together to give the final argument to `TEXIMAGE_PARAMS`. The fields stored here
are:

```
u32 {
    offset_shr_3:    bits(0,16)      (shift left by 3 to get offset for Block1/2; by 2 for Block3)
    w:               bits(20,23)     (8 << w = width in the S-direction)
    h:               bits(23,26)     (8 << h = height in the T-direction)
    format:          bits(26,29)     (texture format)
    color0:          bits(29,30)     (whether color 0 is transparent; palette textures only)
}
```

A `PaletteList` contains `Palettes`.

```
PaletteList {
    NameList(Palette)
}

Palette {
    offset_shr_3:    u16     (shift left by 3 to get the offset into Block4)
    unknown:         u16
}
```