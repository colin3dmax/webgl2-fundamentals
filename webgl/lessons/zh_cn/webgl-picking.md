Title: WebGL2 拾取
Description: 如何在 WebGL 中实现拾取
TOC: 拾取（点击物体）

本文介绍如何使用 WebGL 让用户进行拾取或选择操作。

如果你已经阅读了本站的其他文章，你应该已经意识到 WebGL 本质上只是一个光栅化库。它将三角形、线条和点绘制到画布上，并没有“可选择对象”的概念。它只是通过你提供的着色器输出像素。这意味着“拾取”功能必须由你的代码实现。你需要定义允许用户选择的对象是什么。因此，虽然本文可以介绍一些通用概念，但你需要自行决定如何将这些内容转化为自己应用中的可用概念。


## 点击物体

判断用户点击了哪个物体的最简单方法之一是为每个物体分配一个数字 ID，然后使用该 ID 作为颜色绘制所有物体，不使用光照和纹理。  
这样我们就得到了一张各个物体轮廓的图像，深度缓冲区会帮我们处理遮挡排序。  
接着，我们读取鼠标所在像素的颜色值，即可获得该像素所渲染的物体 ID。

要实现这种技术，需要结合之前的几篇文章。第一篇是关于 [绘制多个物体](webgl-drawing-multiple-things.html) 的文章，因为它演示了如何绘制多个物体，我们可以基于此进行拾取。

此外，我们通常希望将这些 ID 离屏渲染，即通过 [渲染到纹理](webgl-render-to-texture.html) 来实现，因此我们也会添加相关代码。

那么，我们从 [绘制多个物体](webgl-drawing-multiple-things.html) 文章中的最后一个示例，绘制200个物体的开始。

在此基础上，加入一个帧缓冲对象，并附加纹理和深度缓冲区，参照 [渲染到纹理](webgl-render-to-texture.html) 文章中的最后一个示例。

```js
// Create a texture to render to
const targetTexture = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, targetTexture);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

// create a depth renderbuffer
const depthBuffer = gl.createRenderbuffer();
gl.bindRenderbuffer(gl.RENDERBUFFER, depthBuffer);

function setFramebufferAttachmentSizes(width, height) {
  gl.bindTexture(gl.TEXTURE_2D, targetTexture);
  // define size and format of level 0
  const level = 0;
  const internalFormat = gl.RGBA;
  const border = 0;
  const format = gl.RGBA;
  const type = gl.UNSIGNED_BYTE;
  const data = null;
  gl.texImage2D(gl.TEXTURE_2D, level, internalFormat,
                width, height, border,
                format, type, data);

  gl.bindRenderbuffer(gl.RENDERBUFFER, depthBuffer);
  gl.renderbufferStorage(gl.RENDERBUFFER, gl.DEPTH_COMPONENT16, width, height);
}

// Create and bind the framebuffer
const fb = gl.createFramebuffer();
gl.bindFramebuffer(gl.FRAMEBUFFER, fb);

// attach the texture as the first color attachment
const attachmentPoint = gl.COLOR_ATTACHMENT0;
const level = 0;
gl.framebufferTexture2D(gl.FRAMEBUFFER, attachmentPoint, gl.TEXTURE_2D, targetTexture, level);

// make a depth buffer and the same size as the targetTexture
gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.DEPTH_ATTACHMENT, gl.RENDERBUFFER, depthBuffer);
```

我们将设置纹理和深度渲染缓冲区尺寸的代码封装到一个函数中，这样可以方便地调用它来调整尺寸，使其与画布大小匹配。

在渲染代码中，如果画布尺寸发生变化，我们会相应调整纹理和渲染缓冲区的尺寸以保持一致。

```js
function drawScene(time) {
  time *= 0.0005;

-  webglUtils.resizeCanvasToDisplaySize(gl.canvas);
+  if (webglUtils.resizeCanvasToDisplaySize(gl.canvas)) {
+    // the canvas was resized, make the framebuffer attachments match
+    setFramebufferAttachmentSizes(gl.canvas.width, gl.canvas.height);
+  }

...
```

接下来我们需要第二个着色器。  
示例中的着色器是使用的顶点颜色进行渲染，但我们需要一个能够设置为纯色以渲染 ID 的着色器。  
所以，首先这是我们的第二个着色器：

```js
const pickingVS = `#version 300 es
  in vec4 a_position;
  
  uniform mat4 u_matrix;
  
  void main() {
    // Multiply the position by the matrix.
    gl_Position = u_matrix * a_position;
  }
`;

const pickingFS = `#version 300 es
  precision highp float;
  
  uniform vec4 u_id;

  out vec4 outColor;
  
  void main() {
     outColor = u_id;
  }
`;
```

接下来我们需要使用我们的 [辅助函数](webgl-less-code-more-fun.html)  
来编译、链接着色器并查找变量位置。

```js
// setup GLSL program
// note: we need the attribute positions to match across programs
// so that we only need one vertex array per shape
const options = {
  attribLocations: {
    a_position: 0,
    a_color: 1,
  },
};
const programInfo = twgl.createProgramInfo(gl, [vs, fs], options);
const pickingProgramInfo = twgl.createProgramInfo(gl, [pickingVS, pickingFS], options);
```

与本站大多数示例不同的是，这是少数几次需要用两个不同的着色器绘制相同数据的情况之一。因为我们需要确保两个着色器中的属性位置一致。可以通过两种方式实现：其中一种是在 GLSL 中手动设置属性位置。

```glsl
layout (location = 0) in vec4 a_position;
layout (location = 1) in vec4 a_color;
```

另一种方式是在链接着色器程序**之前**调用 `gl.bindAttribLocation`。

```js
gl.bindAttribLocation(someProgram, 0, 'a_position');
gl.bindAttribLocation(someProgram, 1, 'a_color');
gl.linkProgram(someProgram);
```

后一种方式不常见，但它更符合  
[D.R.Y.原则](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)。  
如果我们传入属性名和想要绑定的位置，  
我们的辅助库会帮我们调用 `gl.bindAttribLocation`，  
这就是上面代码的做法。

这样我们就能保证两个程序中 `a_position` 属性都使用位置 0，  
从而能用同一个顶点数组对象配合两个程序。

接下来，我们需要能够对所有物体渲染两次：  
一次使用分配给它们的着色器，另一次使用刚写的这个着色器。  
因此，我们把当前渲染所有物体的代码提取到一个函数中。


```js
function drawObjects(objectsToDraw, overrideProgramInfo) {
  objectsToDraw.forEach(function(object) {
    const programInfo = overrideProgramInfo || object.programInfo;
    const bufferInfo = object.bufferInfo;
    const vertexArray = object.vertexArray;

    gl.useProgram(programInfo.program);

    // Setup all the needed attributes.
    gl.bindVertexArray(vertexArray);

    // Set the uniforms.
    twgl.setUniforms(programInfo, object.uniforms);

    // Draw (calls gl.drawArrays or gl.drawElements)
    twgl.drawBufferInfo(gl, object.bufferInfo);
  });
}
```

`drawObjects` 函数接受一个可选参数 `overrideProgramInfo`，我们可以传入它来使用拾取着色器，替代物体原本分配的着色器。

我们调用该函数两次：一次将物体绘制到带有 ID 的纹理中，另一次将场景绘制到画布上。

```js
// Draw the scene.
function drawScene(time) {
  time *= 0.0005;

  ...

  // Compute the matrices for each object.
  objects.forEach(function(object) {
    object.uniforms.u_matrix = computeMatrix(
        viewProjectionMatrix,
        object.translation,
        object.xRotationSpeed * time,
        object.yRotationSpeed * time);
  });

+  // ------ Draw the objects to the texture --------
+
+  gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
+  gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
+
+  gl.enable(gl.CULL_FACE);
+  gl.enable(gl.DEPTH_TEST);
+
+  // Clear the canvas AND the depth buffer.
+  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
+
+  drawObjects(objectsToDraw, pickingProgramInfo);
+
+  // ------ Draw the objects to the canvas
+
+  gl.bindFramebuffer(gl.FRAMEBUFFER, null);
+  gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);
+
+  drawObjects(objectsToDraw);

  requestAnimationFrame(drawScene);
}
```

我们的拾取着色器需要通过 `u_id` 设置一个 ID，  
因此我们在设置物体时，将该值添加到它们的 uniform 数据中。

```js
// Make infos for each object for each object.
const baseHue = rand(0, 360);
const numObjects = 200;
for (let ii = 0; ii < numObjects; ++ii) {
+  const id = ii + 1;

  // pick a shape
  const shape = shapes[rand(shapes.length) | 0];

  const object = {
    uniforms: {
      u_colorMult: chroma.hsv(eMod(baseHue + rand(0, 120), 360), rand(0.5, 1), rand(0.5, 1)).gl(),
      u_matrix: m4.identity(),
+      u_id: [
+        ((id >>  0) & 0xFF) / 0xFF,
+        ((id >>  8) & 0xFF) / 0xFF,
+        ((id >> 16) & 0xFF) / 0xFF,
+        ((id >> 24) & 0xFF) / 0xFF,
+      ],
    },
    translation: [rand(-100, 100), rand(-100, 100), rand(-150, -50)],
    xRotationSpeed: rand(0.8, 1.2),
    yRotationSpeed: rand(0.8, 1.2),
  };
  objects.push(object);

  // Add it to the list of things to draw.
  objectsToDraw.push({
    programInfo: programInfo,
    bufferInfo: shape.bufferInfo,
    vertexArray: shape.vertexArray,
    uniforms: object.uniforms,
  });
}
```

这是可行的，因为我们的 [辅助库](webgl-less-code-more-fun.html) 会帮我们自动设置 uniform。

我们必须将 ID 拆分到 R、G、B 和 A 四个通道中。  
由于纹理的格式/类型是 `gl.RGBA` 和 `gl.UNSIGNED_BYTE`，  
每个通道只有 8 位，因此最多只能表示 256 个值。  
但通过将 ID 分别存储在 4 个通道中，我们总共可以表示 32 位，  
也就是超过 40 亿个不同的值。

我们对 ID 加 1，是为了将 0 用作“鼠标下方没有物体”的标识。

现在，让我们来高亮鼠标下的物体。

首先需要一些代码来获取相对于画布的鼠标位置。

```js
// mouseX and mouseY are in CSS display space relative to canvas
let mouseX = -1;
let mouseY = -1;

...

gl.canvas.addEventListener('mousemove', (e) => {
   const rect = canvas.getBoundingClientRect();
   mouseX = e.clientX - rect.left;
   mouseY = e.clientY - rect.top;
});
```

注意，上面代码中的 `mouseX` 和 `mouseY` 是以 CSS 像素表示的显示空间坐标。  
也就是说，它们是在画布显示出来的区域中的坐标，而不是画布实际像素空间中的坐标。
换句话说，如果你有一个画布像下面这样的：

```html
<canvas width="11" height="22" style="width:33px; height:44px;"></canvas>
```

那么在该画布上，`mouseX` 的取值范围将是 0 到 33，`mouseY` 的取值范围是 0 到 44。  
更多信息请参阅 [这篇文章](webgl-resizing-the-canvas.html)。

现在我们已经有了鼠标位置，接下来添加一些代码来查找鼠标下方的像素。

```js
const pixelX = mouseX * gl.canvas.width / gl.canvas.clientWidth;
const pixelY = gl.canvas.height - mouseY * gl.canvas.height / gl.canvas.clientHeight - 1;
const data = new Uint8Array(4);
gl.readPixels(
    pixelX,            // x
    pixelY,            // y
    1,                 // width
    1,                 // height
    gl.RGBA,           // format
    gl.UNSIGNED_BYTE,  // type
    data);             // typed array to hold result
const id = data[0] + (data[1] << 8) + (data[2] << 16) + (data[3] << 24);
```

上面的代码中对 `pixelX` 和 `pixelY` 的计算是将显示空间中的 `mouseX` 和 `mouseY` 转换为画布像素空间中的坐标。  
换句话说，以前面的例子为例，当 `mouseX` 范围是 0 到 33，`mouseY` 范围是 0 到 44 时，`pixelX` 的范围将是 0 到 11，`pixelY` 的范围将是 0 到 22。

在实际代码中，我们使用了一个工具函数 `resizeCanvasToDisplaySize`，并将纹理设置为与画布相同的大小，所以显示尺寸和画布尺寸是一致的。不过，我们的代码已经为两者不一致的情况做了准备。

现在我们得到了一个 ID，为了高亮选中的物体，我们需要更改它在画布上渲染时所使用的颜色。  
我们使用的着色器中包含一个 `u_colorMult` 的 uniform，因此我们可以在检测到某个物体被鼠标选中时，先查找该物体，保存它原本的 `u_colorMult` 值，将其替换为选中时的颜色，然后在绘制完后再恢复。

```js
// mouseX and mouseY are in CSS display space relative to canvas
let mouseX = -1;
let mouseY = -1;
+let oldPickNdx = -1;
+let oldPickColor;
+let frameCount = 0;

// Draw the scene.
function drawScene(time) {
  time *= 0.0005;
+  ++frameCount;

  // ------ Draw the objects to the texture --------

  ...

  // ------ Figure out what pixel is under the mouse and read it

  const pixelX = mouseX * gl.canvas.width / gl.canvas.clientWidth;
  const pixelY = gl.canvas.height - mouseY * gl.canvas.height / gl.canvas.clientHeight - 1;
  const data = new Uint8Array(4);
  gl.readPixels(
      pixelX,            // x
      pixelY,            // y
      1,                 // width
      1,                 // height
      gl.RGBA,           // format
      gl.UNSIGNED_BYTE,  // type
      data);             // typed array to hold result
  const id = data[0] + (data[1] << 8) + (data[2] << 16) + (data[3] << 24);

  // restore the object's color
  if (oldPickNdx >= 0) {
    const object = objects[oldPickNdx];
    object.uniforms.u_colorMult = oldPickColor;
    oldPickNdx = -1;
  }

  // highlight object under mouse
  if (id > 0) {
    const pickNdx = id - 1;
    oldPickNdx = pickNdx;
    const object = objects[pickNdx];
    oldPickColor = object.uniforms.u_colorMult;
    object.uniforms.u_colorMult = (frameCount & 0x8) ? [1, 0, 0, 1] : [1, 1, 0, 1];
  }

  // ------ Draw the objects to the canvas

```

这样一来，当我们将鼠标移动到场景中时，鼠标下方的物体就会闪烁显示出来。

{{{example url="../webgl-picking-w-gpu.html" }}}

我们可以进行一个优化，目前我们将 ID 渲染到与画布相同大小的纹理中，  
这在概念上是最简单的方式。

但实际上，我们只需要渲染鼠标下的那个像素即可。要实现这一点，我们可以构造一个视锥（frustum），其数学计算范围仅覆盖那一个像素的空间。

到目前为止，对于 3D 场景我们一直使用的是一个名为 `perspective` 的函数，  
它接收视野角（field of view）、宽高比（aspect ratio）、以及 z 轴的近远平面，  
并生成一个透视投影矩阵，将由这些值定义的视锥转换为裁剪空间（clip space）。

大多数 3D 数学库还有另一个名为 `frustum` 的函数，  
它接收 6 个参数：近裁剪面上的 left、right、top、bottom 值，  
以及 zNear 和 zFar 两个 z 轴平面，并据此生成一个透视投影矩阵。

利用这个函数，我们可以生成一个只覆盖鼠标下那一个像素的透视矩阵。

首先，如果我们使用 `perspective` 函数，计算出其近裁剪面在视图空间中的边缘位置和尺寸。

```js
// compute the rectangle the near plane of our frustum covers
const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
const top = Math.tan(fieldOfViewRadians * 0.5) * near;
const bottom = -top;
const left = aspect * bottom;
const right = aspect * top;
const width = Math.abs(right - left);
const height = Math.abs(top - bottom);
```

因此，`left`、`right`、`width` 和 `height` 表示近裁剪面的尺寸和位置。  
现在在这个平面上，可以计算出鼠标下方那一个像素的尺寸和位置，然后将这些值传入 `frustum` 函数，以生成一个只覆盖该像素的投影矩阵。

```js
// compute the portion of the near plane covers the 1 pixel
// under the mouse.
const pixelX = mouseX * gl.canvas.width / gl.canvas.clientWidth;
const pixelY = gl.canvas.height - mouseY * gl.canvas.height / gl.canvas.clientHeight - 1;

const subLeft = left + pixelX * width / gl.canvas.width;
const subBottom = bottom + pixelY * height / gl.canvas.height;
const subWidth = width / gl.canvas.width;
const subHeight = height / gl.canvas.height;

// make a frustum for that 1 pixel
const projectionMatrix = m4.frustum(
    subLeft,
    subLeft + subWidth,
    subBottom,
    subBottom + subHeight,
    near,
    far);
```

要使用这种方法，我们需要做一些修改。目前我们的着色器只接受一个 `u_matrix`，这意味着如果想用不同的投影矩阵绘制，我们必须在每一帧对每个物体计算两次矩阵。一次用于正常的画布投影矩阵，另一次用于这个单像素投影矩阵。

我们可以将这个责任从 JavaScript 中移除，通过把矩阵乘法移动到顶点着色器中实现。

```html
const vs = `#version 300 es

in vec4 a_position;
in vec4 a_color;

-uniform mat4 u_matrix;
+uniform mat4 u_viewProjection;
+uniform mat4 u_world;

out vec4 v_color;

void main() {
  // Multiply the position by the matrix.
-  gl_Position = u_matrix * a_position;
+  gl_Position = u_viewProjection * u_world * a_position;

  // Pass the color to the fragment shader.
  v_color = a_color;
}
`;

...

const pickingVS = `#version 300 es
  in vec4 a_position;
  
-  uniform mat4 u_matrix;
+  uniform mat4 u_viewProjection;
+  uniform mat4 u_world;
  
  void main() {
    // Multiply the position by the matrix.
-   gl_Position = u_matrix * a_position;
+    gl_Position = u_viewProjection * u_world * a_position;
  }
`;
```

这样一来，我们就可以让 JavaScript 中的 `viewProjectionMatrix` 在所有物体间共享使用。

```js
const objectsToDraw = [];
const objects = [];
+const viewProjectionMatrix = m4.identity();

// Make infos for each object for each object.
const baseHue = rand(0, 360);
const numObjects = 200;
for (let ii = 0; ii < numObjects; ++ii) {
  const id = ii + 1;

  // pick a shape
  const shape = shapes[rand(shapes.length) | 0];

  const object = {
    uniforms: {
      u_colorMult: chroma.hsv(eMod(baseHue + rand(0, 120), 360), rand(0.5, 1), rand(0.5, 1)).gl(),
-      u_matrix: m4.identity(),
+      u_world: m4.identity(),
+      u_viewProjection: viewProjectionMatrix,
      u_id: [
        ((id >>  0) & 0xFF) / 0xFF,
        ((id >>  8) & 0xFF) / 0xFF,
        ((id >> 16) & 0xFF) / 0xFF,
        ((id >> 24) & 0xFF) / 0xFF,
      ],
    },
    translation: [rand(-100, 100), rand(-100, 100), rand(-150, -50)],
    xRotationSpeed: rand(0.8, 1.2),
    yRotationSpeed: rand(0.8, 1.2),
  };
```

在计算每个物体的矩阵时，我们不再需要将视图投影矩阵包含在内。

```js
-function computeMatrix(viewProjectionMatrix, translation, xRotation, yRotation) {
-  let matrix = m4.translate(viewProjectionMatrix,
+function computeMatrix(translation, xRotation, yRotation) {
+  let matrix = m4.translation(
      translation[0],
      translation[1],
      translation[2]);
  matrix = m4.xRotate(matrix, xRotation);
  return m4.yRotate(matrix, yRotation);
}
...

// Compute the matrices for each object.
objects.forEach(function(object) {
  object.uniforms.u_world = computeMatrix(
-      viewProjectionMatrix,
      object.translation,
      object.xRotationSpeed * time,
      object.yRotationSpeed * time);
});
```

我们将创建一个仅有 1×1 像素的纹理和深度缓冲区。

```js
setFramebufferAttachmentSizes(1, 1);

...

// Draw the scene.
function drawScene(time) {
  time *= 0.0005;
  ++frameCount;

-  if (webglUtils.resizeCanvasToDisplaySize(gl.canvas)) {
-    // the canvas was resized, make the framebuffer attachments match
-    setFramebufferAttachmentSizes(gl.canvas.width, gl.canvas.height);
-  }
+  webglUtils.resizeCanvasToDisplaySize(gl.canvas);
```

然后，在渲染离屏 ID 之前，我们将使用 1 像素的投影矩阵设置视图投影，  
而在绘制到画布时，我们使用原始的投影矩阵。

```js
-// Compute the projection matrix
-const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
-const projectionMatrix =
-    m4.perspective(fieldOfViewRadians, aspect, 1, 2000);

// Compute the camera's matrix using look at.
const cameraPosition = [0, 0, 100];
const target = [0, 0, 0];
const up = [0, 1, 0];
const cameraMatrix = m4.lookAt(cameraPosition, target, up);

// Make a view matrix from the camera matrix.
const viewMatrix = m4.inverse(cameraMatrix);

-const viewProjectionMatrix = m4.multiply(projectionMatrix, viewMatrix);

// Compute the matrices for each object.
objects.forEach(function(object) {
  object.uniforms.u_world = computeMatrix(
      object.translation,
      object.xRotationSpeed * time,
      object.yRotationSpeed * time);
});

// ------ Draw the objects to the texture --------

// Figure out what pixel is under the mouse and setup
// a frustum to render just that pixel

{
  // compute the rectangle the near plane of our frustum covers
  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
  const top = Math.tan(fieldOfViewRadians * 0.5) * near;
  const bottom = -top;
  const left = aspect * bottom;
  const right = aspect * top;
  const width = Math.abs(right - left);
  const height = Math.abs(top - bottom);

  // compute the portion of the near plane covers the 1 pixel
  // under the mouse.
  const pixelX = mouseX * gl.canvas.width / gl.canvas.clientWidth;
  const pixelY = gl.canvas.height - mouseY * gl.canvas.height / gl.canvas.clientHeight - 1;

  const subLeft = left + pixelX * width / gl.canvas.width;
  const subBottom = bottom + pixelY * height / gl.canvas.height;
  const subWidth = width / gl.canvas.width;
  const subHeight = height / gl.canvas.height;

  // make a frustum for that 1 pixel
  const projectionMatrix = m4.frustum(
      subLeft,
      subLeft + subWidth,
      subBottom,
      subBottom + subHeight,
      near,
      far);
+  m4.multiply(projectionMatrix, viewMatrix, viewProjectionMatrix);
}

gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
gl.viewport(0, 0, 1, 1);

gl.enable(gl.CULL_FACE);
gl.enable(gl.DEPTH_TEST);

// Clear the canvas AND the depth buffer.
gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

drawObjects(objectsToDraw, pickingProgramInfo);

// read the 1 pixel
-const pixelX = mouseX * gl.canvas.width / gl.canvas.clientWidth;
-const pixelY = gl.canvas.height - mouseY * gl.canvas.height / gl.canvas.clientHeight - 1;
const data = new Uint8Array(4);
gl.readPixels(
-    pixelX,            // x
-    pixelY,            // y
+    0,                 // x
+    0,                 // y
    1,                 // width
    1,                 // height
    gl.RGBA,           // format
    gl.UNSIGNED_BYTE,  // type
    data);             // typed array to hold result
const id = data[0] + (data[1] << 8) + (data[2] << 16) + (data[3] << 24);

// restore the object's color
if (oldPickNdx >= 0) {
  const object = objects[oldPickNdx];
  object.uniforms.u_colorMult = oldPickColor;
  oldPickNdx = -1;
}

// highlight object under mouse
if (id > 0) {
  const pickNdx = id - 1;
  oldPickNdx = pickNdx;
  const object = objects[pickNdx];
  oldPickColor = object.uniforms.u_colorMult;
  object.uniforms.u_colorMult = (frameCount & 0x8) ? [1, 0, 0, 1] : [1, 1, 0, 1];
}

// ------ Draw the objects to the canvas

+{
+  // Compute the projection matrix
+  const aspect = gl.canvas.clientWidth / gl.canvas.clientHeight;
+  const projectionMatrix =
+      m4.perspective(fieldOfViewRadians, aspect, near, far);
+
+  m4.multiply(projectionMatrix, viewMatrix, viewProjectionMatrix);
+}

gl.bindFramebuffer(gl.FRAMEBUFFER, null);
gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);

drawObjects(objectsToDraw);
```

您可以看到数学计算是有效的，我们只是绘制了一个像素，但依然能够准确地确定鼠标下方的物体。

{{{example url="../webgl-picking-w-gpu-1pixel.html"}}}


