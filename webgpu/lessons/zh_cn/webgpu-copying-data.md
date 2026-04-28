Title: WebGPU 数据拷贝
Description: 在缓冲区和纹理之间拷贝数据
TOC: 数据拷贝

在大多数文章中，我们使用 `writeBuffer` 函数将数据写入缓冲区，使用 `writeTexture` 将数据写入纹理。将数据放入缓冲区或纹理有多种方法。

## `writeBuffer`

`writeBuffer` 将数据从 JavaScript 中的 `TypedArray` 或 `ArrayBuffer` 拷贝到缓冲区。这可以说是将数据放入缓冲区最直接的方式。

`writeBuffer` 的格式如下：

```js
device.queue.writeBuffer(
  destBuffer,  // 要写入的缓冲区
  destOffset,  // 在目标缓冲区中开始写入的位置
  srcData,     // typedArray 或 arrayBuffer
  srcOffset?,  // **元素**为单位的 srcData 中的起始偏移量
  size?,       // 要拷贝的 srcData 的**元素**数量
)
```

如果未传入 `srcOffset`，则默认为 `0`。如果未传入 `size`，则默认为 `srcData` 的大小。

> 重要：`srcOffset` 和 `size` 是 `srcData` 中的**元素**数量
>
> 换句话说，
>
> ```js
> device.queue.writeBuffer(
>   someBuffer,
>   someOffset,
>   someFloat32Array,
>   6,
>   7,
> )
> ```
>
> 上面的代码将从 float32 #6 开始，拷贝 7 个 float32 的数据。
> 换句话说，它将从 `someFloat32Array` 所视图的 arrayBuffer 部分中，从字节 24 开始拷贝 28 字节。

## `writeTexture`

`writeTexture` 将数据从 JavaScript 中的 `TypedArray` 或 `ArrayBuffer` 拷贝到纹理。

`writeTexture` 的签名如下：

```js
device.queue.writeTexture(
  // 目标详细信息
  { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // 源数据
  srcData,

  // 源数据详细信息
  { offset: 0, bytesPerRow, rowsPerImage },

  // 大小：
  [ width, height, depthOrArrayLayers ] 或 { width, height, depthOrArrayLayers }
)
```

注意事项：

* `texture` 必须具有 `GPUTextureUsage.COPY_DST` 用法标志

* `mipLevel`、`origin` 和 `aspect` 都有默认值，所以通常不需要指定

* `bytesPerRow`：这是前进到下一个*块行*数据需要跳过的字节数。

  如果你要拷贝超过 1 个*块行*，这是必需的。几乎
  总是要拷贝超过 1 个*块行*的数据，因此
  几乎总是需要这个参数。

* `rowsPerImage`：这是从一张图像的开始到下一张图像开始需要跳过的*块行*数量。

  如果你要拷贝超过 1 层，则这是必需的。换句话说，
  如果 size 参数中的 `depthOrArrayLayers` > 1，则你需要提供此值。

你可以将拷贝过程想象成这样：

```js
   // 伪代码
   const [x, y, z] = origin ?? [0, 0, 0];
   const [blockWidth, blockHeight, bytesPerBlock] =
      getBlockInfoForTextureFormat(texture.format);

   const blocksAcross = width / blockWidth;
   const blocksDown = height / blockHeight;
   const bytesPerBlockRow = blocksAcross * bytesPerBlock;

   for (layer = 0; layer < depthOrArrayLayers; layer) {
      for (row = 0; row < blocksDown; ++row) {
        const start = offset + (layer * rowsPerImage + row) * bytesPerRow;
        copyRowToTexture(
            texture,               // 要拷贝到哪个纹理
            x, y + row, z + layer, // 拷贝到纹理中的位置
            srcDataAsBytes + start,
            bytesPerBlockRow);
      }
   }
```

### <a id="a-block-rows"></a>**块行（block row）**

纹理被组织成块。对于大多数*常规*纹理，块宽度和块高度都是 1。对于压缩纹理，情况会发生变化。例如，格式 `bc1-rgba-unorm` 的块宽度为 4，块高度为 4。这意味着如果你设置宽度为 8，高度为 12，只会拷贝 6 个块。第一行 2 个块，第二行 2 个块，第三行 2 个块。

对于压缩纹理，size 和 origin 必须与块大小对齐。

> 重要：WebGPU 中任何接受大小（定义为 `GPUExtent3D`）的地方都可以是 1 到 3 个数字的数组，也可以是具有 1 到 3 个属性的对象。`height` 和 `depthOrArrayLayers` 默认为 1，因此
>
> * `[2]` 大小：width = 2, height = 1, depthOrArrayLayers = 1
> * `[2, 3]` 大小：width = 2, height = 3, depthOrArrayLayers = 1
> * `[2, 3, 4]` 大小：width = 2, height = 3, depthOrArrayLayers = 4
> * `{ width: 2 }` 大小：width = 2, height = 1, depthOrArrayLayers = 1
> * `{ width: 2, height: 3 }` 大小：width = 2, height = 3, depthOrArrayLayers = 1
> * `{ width: 2, height: 3, depthOrArrayLayers: 4 }` 大小：width = 2, height = 3, depthOrArrayLayers = 4

> 同样，任何出现 origin 的地方（默认为 `GPUOrigin3D`），你可以使用 3 个数字的数组，或具有 `x`、`y`、`z` 属性的对象。它们都默认为 0，因此
>
> * `[5]` origin：x = 5, y = 0, z = 0
> * `[5, 6]` origin：x = 5, y = 6, z = 0
> * `[5, 6, 7]` origin：x = 5, y = 6, z = 7
> * `{ x: 5 }` origin：x = 5, y = 0, z = 0
> * `{ x: 5, y: 6 }` origin：x = 5, y = 6, z = 0
> * `{ x: 5, y: 6, z: 7 }` origin：x = 5, y = 6, z = 7

* `aspect` 主要在拷贝深度模具格式数据时才起作用。你一次只能拷贝到一个方面，即 `depth-only` 或 `stencil-only`。

> 小知识：纹理上有 `width`、`height` 和 `depthOrArrayLayers` 属性，这意味着它是一个有效的 `GPUExtent3D`。换句话说，给定这个纹理
>
> ```js
> const texture = device.createTexture({
>   format: 'r8unorm',
>   size: [2, 4],
>   usage: GPUTextureUsage.COPY_DST | GPUTextureUsage.TEXTURE_ATTACHMENT,
> });
> ```
>
> 以下所有用法都是有效的
>
> ```js
> // 拷贝 2x4 像素的数据到纹理
> const bytesPerRow = 2;
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, [2, 4]);
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, [texture.width, texture.height]);
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, {width: 2, height: 4});
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, {width: texture.width, height: texture.height});
> device.queue.writeTexture({ texture }, data, { bytesPerRow }, texture); // !!!
> ```
>
> 最后一个有效是因为纹理有 `width`、`height` 和 `depthOrArrayLayers`。我们没有使用这种风格，因为它不太清晰，但它是有效的。

## `copyBufferToBuffer`

顾名思义，`copyBufferToBuffer` 将数据从一个缓冲区拷贝到另一个缓冲区。

签名：

```js
encoder.copyBufferToBuffer(
  source,       // 要拷贝的源缓冲区
  sourceOffset, // 开始拷贝的位置
  dest,         // 目标缓冲区
  destOffset,   // 开始拷贝到的位置
  size,         // 要拷贝的字节数
)
```

* `source` 必须具有 `GPUBufferUsage.COPY_SRC` 用法标志
* `dest` 必须具有 `GPUBufferUsage.COPY_DST` 用法标志
* `size` 必须是 4 的倍数

## `copyBufferToTexture`

顾名思义，`copyBufferToTexture` 将数据从缓冲区拷贝到纹理。

签名：

```js
encoder.copyBufferToTexture(
  // 源缓冲区详细信息
  { buffer, offset: 0, bytesPerRow, rowsPerImage },

  // 目标纹理详细信息
  { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // 大小：
  [ width, height, depthOrArrayLayers ] 或 { width, height, depthOrArrayLayers }
)
```

这与 `writeTexture` 的参数几乎完全相同。最大的区别是 `bytesPerRow` **必须是 256 的倍数！！**

* `texture` 必须具有 `GPUTextureUsage.COPY_DST` 用法标志
* `buffer` 必须具有 `GPUBufferUsage.COPY_SRC` 用法标志

## `copyTextureToBuffer`

顾名思义，`copyTextureToBuffer` 将数据从纹理拷贝到缓冲区。

签名：

```js
encoder.copyTextureToBuffer(
  // 源纹理详细信息
  { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // 目标缓冲区详细信息
  { buffer, offset: 0, bytesPerRow, rowsPerImage },

  // 大小：
  [ width, height, depthOrArrayLayers ] 或 { width, height, depthOrArrayLayers }
)
```

这与 `copyBufferToTexture` 的参数类似，只是纹理（现在是源）和缓冲区（现在是目标）交换了位置。与 `copyBufferToTexture` 一样，`bytesPerRow` **必须是 256 的倍数！！**

* `texture` 必须具有 `GPUTextureUsage.COPY_SRC` 用法标志
* `buffer` 必须具有 `GPUBufferUsage.COPY_DST` 用法标志

## `copyTextureToTexture`

`copyTextureToTexture` 将部分纹理拷贝到另一个纹理。

两个纹理必须是相同格式，或者只能通过 `'-srgb'` 后缀不同。

签名：

```js
encoder.copyTextureToTexture(
  // 源纹理详细信息
  src: { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // 目标纹理详细信息
  dst: { texture, mipLevel: 0, origin: [0, 0, 0], aspect: "all" },

  // 大小：
  [ width, height, depthOrArrayLayers ] 或 { width, height, depthOrArrayLayers }
)
```

* src.`texture` 必须具有 `GPUTextureUsage.COPY_SRC` 用法标志
* dst.`texture` 必须具有 `GPUTextureUsage.COPY_DST` 用法标志
* `width` 必须是块宽度的倍数
* `height` 必须是块高度的倍数
* src.`origin[0]` 或 `.x` 必须是块宽度的倍数
* src.`origin[1]` 或 `.y` 必须是块高度的倍数
* dst.`origin[0]` 或 `.x` 必须是块宽度的倍数
* dst.`origin[1]` 或 `.y` 必须是块高度的倍数

## 着色器

着色器可以读取和写入存储缓冲区、存储纹理，并且间接地可以渲染到纹理。这些都是将数据放入缓冲区和纹理的方法。换句话说，你可以编写着色器来生成、拷贝和传输数据。

## 映射缓冲区

你可以映射一个缓冲区。映射缓冲区意味着使它可以被 JavaScript 读取或写入。至少在 WebGPU 第一版中，可映射缓冲区有严格限制，即：可映射缓冲区只能用作临时的拷贝源或拷贝目标。可映射缓冲区不能用作任何其他类型的缓冲区（如 uniform 缓冲区、顶点缓冲区、索引缓冲区、存储缓冲区等）[^mappedAtCreation]

[^mappedAtCreation]: 例外情况是如果你设置了 `mappedAtCreation: true`。请参阅 [mappedAtCreation](#a-mapped-at-creation)。

你可以用 2 种用法标志组合创建可映射缓冲区：

* `GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST`

  这是一个缓冲区，你可以使用上面的拷贝命令从另一个缓冲区或纹理拷贝数据，然后映射它以在 JavaScript 中读取值

* `GPUBufferUsage.MAP_WRITE | GPUBufferUsage.COPY_SRC`

  这是一个你可以在 JavaScript 中映射的缓冲区，你可以在其中放入数据，最后取消映射并使用上面的拷贝命令将其内容拷贝到另一个缓冲区或纹理

映射缓冲区的过程是异步的。你调用 `buffer.mapAsync(mode, offset = 0, size?)`，其中 `offset` 和 `size` 以字节为单位。如果未指定 `size`，则为整个缓冲区的大小。`mode` 必须是 `GPUMapMode.READ` 或 `GPUMapMode.WRITE`，当然必须与你创建缓冲区时传入的 `MAP_` 用法标志匹配。

`mapAsync` 返回一个 `Promise`。当 promise 解析时，缓冲区就可以映射了。你可以通过调用 `buffer.getMappedRange(offset = 0, size?)` 来查看缓冲区的部分或全部内容，其中 `offset` 是映射部分缓冲区的字节偏移量。`getMappedRange` 返回一个 `ArrayBuffer`，因此通常你需要用它来构造 TypedArray 才能使用。

下面是一个映射缓冲区的示例：

```js
const buffer = device.createBuffer({
  size: 1024,
  usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST,
});

// 映射整个缓冲区
await buffer.mapAsync(GPUMapMode.READ);

// 将整个缓冲区作为 32 位浮点数数组获取
const f32 = new Float32Array(buffer.getMappedRange())

...

buffer.unmap();
```

注意：一旦映射，在调用 `unmap` 之前，缓冲区不能被 WebGPU 使用。一旦调用 `unmap`，缓冲区就会从 JavaScript 中消失。换句话说，以这个例子为例：

```js
const f32 = new Float32Array(buffer.getMappedRange())

f32[0] = 123;
console.log(f32[0]); // 打印 123

buffer.unmap();

console.log(f32[0]); // 打印 undefined
```

我们已经在[第一篇文章](webgpu-fundamentals.html#a-run-computations-on-the-gpu)中看到过映射缓冲区读取的示例，我们在存储缓冲区中将一些数字翻倍，然后将结果拷贝到可映射缓冲区并映射它以读取结果。

另一个示例是[计算着色器基础文章](webgpu-compute-shaders.md)，我们将各种 `@builtin` 计算着色器值输出到存储缓冲区，然后将结果拷贝到可映射缓冲区并映射它以读取结果。

## <a id="a-mapped-at-creation"></a>mappedAtCreation

`mappedAtCreation: true` 是创建缓冲区时可以添加的标志。在这种情况下，缓冲区不需要 `GPUBufferUsage.COPY_DST` 或 `GPUBufferUsage.MAP_WRITE` 用法标志。

这是一个特殊的标志，仅用于让你在创建时将数据放入缓冲区。你在创建缓冲区时添加标志 `mappedAtCreation: true`。缓冲区被创建时已经映射好用于写入。例如：

```js
 const buffer = device.createBuffer({
   size: 16,
   usage: GPUBufferUsage.UNIFORM,
   mappedAtCreation: true,
 });
 const arrayBuffer = buffer.getMappedRange(0, buffer.size);
 const f32 = new Float32Array(arrayBuffer);
 f32.set([1, 2, 3, 4]);
 buffer.unmap();
```

或者更简洁：

```js
 const buffer = device.createBuffer({
   size: 16,
   usage: GPUBufferUsage.UNIFORM,
   mappedAtCreation: true,
 });
 new Float32Array(buffer.getMappedRange(0, buffer.size)).set([1, 2, 3, 4]);
 buffer.unmap();
```

请注意，用 `mappedAtCreation: true` 创建的缓冲区不会自动设置任何标志。这只是在首次创建时将数据放入缓冲区的便利方式。它在创建时映射，在你第一次取消映射后，表现得像任何其他缓冲区一样，只会对你指定的用法有效。换句话说，如果你想在以后拷贝到它，需要 `GPUBufferUsage.COPY_DST`；如果你想在以后映射它，需要 `GPUBufferUsage.MAP_READ` 或 `GPUBufferUsage.MAP_WRITE`。

## <a id="a-efficient"></a> 高效使用可映射缓冲区

上面我们看到映射缓冲区是异步的。这意味着从我们通过调用 `mapAsync` 请求缓冲区映射，到缓冲区被映射并可以调用 `getMappedRange` 之间，有一段不确定的时间。

一个常见的解决方法是始终保持一组缓冲区处于已映射状态。因为它们已经映射好了，所以可以立即使用。一旦你使用了一个并取消映射，并且只要你提交了使用该缓冲区的命令，你就可以再次请求映射它。当它的 promise 解析时，你把它放回已映射缓冲池中。如果你需要可映射缓冲区但没有可用的，你就创建一个新的并添加到池中。
