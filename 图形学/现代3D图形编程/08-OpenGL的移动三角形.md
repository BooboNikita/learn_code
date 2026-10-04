# OpenGL 的移动三角形

本教程讨论如何让物体动起来，其中会引入若干新的着色器技巧。

## 移动顶点

人们最容易想到的移动三角形或其他物体的办法，就是直接修改顶点位置数据。在前面的教程中我们已经学到，顶点数据存放在缓冲对象中。因此任务就是修改缓冲对象中的顶点数据。`CPUPositionOffset.cpp` 做的正是这件事。

修改分两步完成。第一步是生成将要施加到每个位置上的 X、Y 偏移量。第二步是把该偏移量施加到每个顶点位置上。偏移量的生成由 `ComputePositionOffset` 函数完成：

**示例：位置偏移的计算**

```cpp
void ComputePositionOffsets(float &fXOffset, float &fYOffset)
{
    const float fLoopDuration = 5.0f;
    const float fScale = 3.14159f * 2.0f / fLoopDuration;
    
    float fElapsedTime = glutGet(GLUT_ELAPSED_TIME) / 1000.0f;
    
    float fCurrTimeThroughLoop = fmodf(fElapsedTime, fLoopDuration);
    
    fXOffset = cosf(fCurrTimeThroughLoop * fScale) * 0.5f;
    fYOffset = sinf(fCurrTimeThroughLoop * fScale) * 0.5f;
}
```

这个函数以一个循环为单位计算偏移量。这些偏移量产生圆周运动，并且每 5 秒（由 `fLoopDuration` 控制）回到一次圆周的起点。`glutGet(GLUT_ELAPSED_TIME)` 函数获取自应用程序启动以来经过的整数毫秒数。`fmodf` 函数计算时间的浮点取模。用通俗的话说，它取第一个参数，返回它除以第二个参数所得的余数。因此它返回的是 [0,`fLoopDuration`) 区间内的值，这正是我们制造周期性重复图案所需要的。

`cosf` 和 `sinf` 函数分别计算余弦和正弦。没必要确切知道这两个函数如何工作，它们实际上生成的是直径为 2 的圆。再乘以 0.5f，就把圆缩小到直径为 1。

偏移量计算出来之后，还必须把它们加到顶点数据上。这由 `AdjustVertexData` 函数完成：

**示例：调整顶点数据**

```cpp
void AdjustVertexData(float fXOffset, float fYOffset)
{
    std::vector<float> fNewData(ARRAY_COUNT(vertexPositions));
    memcpy(&fNewData[0], vertexPositions, sizeof(vertexPositions));
    
    for(int iVertex = 0; iVertex < ARRAY_COUNT(vertexPositions); iVertex += 4)
    {
        fNewData[iVertex] += fXOffset;
        fNewData[iVertex + 1] += fYOffset;
    }
    
    glBindBuffer(GL_ARRAY_BUFFER, positionBufferObject);
    glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(vertexPositions), &fNewData[0]);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}
```

这个函数的做法是先把顶点数据复制进一个 `std::vector`，然后把偏移量施加到每个顶点的 X、Y 坐标上。最后三行才是与 OpenGL 相关的部分。

首先，把存放位置数据的缓冲对象绑定到上下文。然后调用新函数 `glBufferSubData`，把这份数据传送到缓冲对象中。

`glBufferData` 与 `glBufferSubData` 的区别在于，SubData 函数*不分配*内存。`glBufferData` 明确分配指定大小的内存；`glBufferSubData` 只是把数据传送到已经存在的内存中。对一个已经分配过的缓冲对象调用 `glBufferData`，等于告诉 OpenGL *重新分配*这块内存，丢弃原有数据并另分配一块全新的内存。反过来，对一个尚未通过 `glBufferData` 分配内存的缓冲对象调用 `glBufferSubData`，则是一个错误。

可以把 `glBufferData` 看作 `malloc` 与 `memcpy` 的组合，而 `glBufferSubData` 只是 `memcpy`。

`glBufferSubData` 函数只能更新缓冲对象内存的一部分。该函数的第二个参数是开始复制位置在缓冲对象内的字节偏移，第三个参数是要复制的字节数。第四个参数是要被复制到缓冲对象该位置的字节数组。

函数的最后一行只是给缓冲对象解绑。这并非严格必要，但用完绑定之后顺手清理是好习惯。

**缓冲对象使用提示**

我们每次绘制东西都会改动缓冲对象的数据。OpenGL 有一种方式可以告诉它你打算做这类事情，这正是 `glBufferData` 最后一个参数的用途。本教程稍微改动了一下缓冲对象的分配方式，把：

```cpp
glBufferData(GL_ARRAY_BUFFER, sizeof(vertexPositions), vertexPositions, GL_STATIC_DRAW);
```

换成了：

```cpp
glBufferData(GL_ARRAY_BUFFER, sizeof(vertexPositions), vertexPositions, GL_STREAM_DRAW);
```

`GL_STATIC_DRAW` 告诉 OpenGL，你打算只设置该缓冲对象中的数据一次。`GL_STREAM_DRAW` 则告诉 OpenGL，你打算持续不断地设置这份数据，通常是每帧一次。这些参数对 API 本身而言*没有任何*意义；它们只是给 OpenGL 实现的提示。恰当使用这些提示，对频繁改动数据时获得良好的缓冲对象性能可能至关重要。后面我们还会看到更多这类提示。

现在的渲染函数变成了这样：

**示例：更新并绘制顶点数据**

```cpp
void display()
{
    float fXOffset = 0.0f, fYOffset = 0.0f;
    ComputePositionOffsets(fXOffset, fYOffset);
    AdjustVertexData(fXOffset, fYOffset);
    
    glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
    glClear(GL_COLOR_BUFFER_BIT);
    
    glUseProgram(theProgram);
    
    glBindBuffer(GL_ARRAY_BUFFER, positionBufferObject);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);
    
    glDrawArrays(GL_TRIANGLES, 0, 3);
    
    glDisableVertexAttribArray(0);
    glUseProgram(0);
    
    glutSwapBuffers();
    glutPostRedisplay();
}
```

前三行获取偏移量并设置顶点数据。除了最后一行之外，其余部分与第一个教程完全相同。函数最后一行用来告诉 FreeGLUT 持续不断地调用 `display`。通常情况下，`display` 只在窗口尺寸变化或窗口被重新显露时才会被调用。`glutPostRedisplay` 让 FreeGLUT 再次调用 `display`。不是立刻调用，但相当快。

运行这个教程，你会看到一个较小的三角形（本教程中尺寸被缩小了）沿着圆周滑动。

## 更好的办法

对于只有 3 个顶点的例子来说，这样做没问题。但设想一个涉及数百万个顶点的场景（在高端应用中，这可不是夸张）。用这种方式移动物体意味着必须从原始顶点数据中复制数百万个顶点，给每一个加上偏移量，然后把这些数据上传到 OpenGL 缓冲对象。而这一切还都发生在渲染*之前*。显然应该有更好的办法；游戏不可能每帧都这么做还保持不错的帧率。

事实上，在相当长一段时间里，游戏确实就是这么做的。在 GeForce 256 之前的日子里，所有游戏都是这样工作的。图形硬件只是接收一份裁剪空间的顶点列表，把它们光栅化成片元和像素。诚然，在那个年代，我们谈论的也许只是每帧 10,000 个三角形。虽然 CPU 此后进步巨大，但它并没有随着图形场景复杂度的增长而同步扩展。

GeForce 256（注意：不是 GT 2xx 系列，而是第一款 GeForce 显卡）是第一块真正做了某种顶点处理的显卡。它可以把顶点存放在 GPU 内存中、读出它们、对它们做某种变换，然后再让它们穿过管线的其余阶段。老式 GeForce 256 能做的那些变换相当有用，但也相当简单。

有了现代硬件和 OpenGL 3.x，我们手上有了一个灵活得多的东西：顶点着色器。

还记得我们正在做什么吗。我们计算一个偏移量。然后把这个偏移量施加到每个顶点位置上。顶点着色器会拿到每个顶点位置。因此，把偏移量直接交给顶点着色器、让它去算出最终的顶点位置显然是合理的，因为它本来就是逐顶点操作的。`VertPositionOffset.cpp` 做的正是这件事。

顶点着色器位于 `data\positionOffset.vert`。这里用到的顶点着色器如下：

**示例：带偏移的顶点着色器**

```glsl
#version 330

layout(location = 0) in vec4 position;
uniform vec2 offset;

void main()
{
    vec4 totalOffset = vec4(offset.x, offset.y, 0.0, 0.0);
    gl_Position = position + totalOffset;
}
```

在定义了输入 `position` 之后，这个着色器又定义了一个 2 维向量 `offset`。但它是用 `uniform` 这个词来定义的，而不是 `in` 或 `out`。这有特定的含义。

**着色器与粒度**

回想一下，着色器每次执行时，被定义为 `in` 的变量都会获得新的值。顶点着色器每次被调用，得到的输入集合都不同于上一次，它是从顶点属性数组和缓冲中读来的。这对顶点位置数据来说很有用，但对我们的偏移量却不是我们想要的。我们希望每个顶点都使用*同一个*偏移量；可以说，是一个"uniform"的偏移量。

定义为 `uniform` 的变量，其变化频率与定义为 `in` 的变量不同。输入变量在着色器每次执行时都会变。uniform 变量（称为 uniform）只在两次渲染调用之间才可能变化。即便如此，它们也只会在用户显式把它们设为新值时才变化。

顶点着色器的输入来自顶点属性数组定义和缓冲对象。相比之下，uniform 是直接设在程序对象上的。

要在程序中设置一个 uniform，我们需要两样东西。第一样是 uniform 的位置。和属性类似，有一个索引指向某个具体的 uniform 值。与属性不同的是，这个位置不能由你自己设定，必须查询得到。本教程中，这是在 `InitializeProgram` 函数里用这一行完成的：

```cpp
offsetLocation = glGetUniformLocation(theProgram, "offset");
```

`glGetUniformLocation` 函数获取第二个参数所命名的 uniform 的位置。注意，即便某个 uniform 在着色器中定义了，GLSL 也*不一定*要为它提供位置。只有当该 uniform 确实在程序中被用到时，它才会有位置，就像我们在顶点着色器中看到的那样；如果该 uniform 没有位置，`glGetUniformLocation` 会返回 -1。

拿到 uniform 位置之后，就可以设置该 uniform 的值了。不过与获取 uniform 位置不同，设置 uniform 的值要求程序当前正处于使用状态，这要通过 `glUseProgram` 完成。因此渲染代码如下：

**示例：带着计算出的偏移量进行绘制**

```cpp
void display()
{
    float fXOffset = 0.0f, fYOffset = 0.0f;
    ComputePositionOffsets(fXOffset, fYOffset);
    
    glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
    glClear(GL_COLOR_BUFFER_BIT);
    
    glUseProgram(theProgram);
    
    glUniform2f(offsetLocation, fXOffset, fYOffset);
    
    glBindBuffer(GL_ARRAY_BUFFER, positionBufferObject);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);
    
    glDrawArrays(GL_TRIANGLES, 0, 3);
    
    glDisableVertexAttribArray(0);
    glUseProgram(0);
    
    glutSwapBuffers();
    glutPostRedisplay();
}
```

我们用 `ComputePositionOffsets` 取得偏移量，再用 `glUniform2f` 设置 uniform 的值。缓冲对象的数据从未被改动；着色器只是承担了繁重的工作。这本来就是着色器阶段存在的意义。

**▌OpenGL 标准命名约定**

> `glUniform2f` 函数用到 OpenGL 命名约定中一个特别的部分，具体来说就是 "2f" 这一段。
>
> uniform 可以有不同类型。又因为 OpenGL 是按 C 而不是 C++ 定义的，所以不存在函数重载的概念。因此，做同一件事但接受不同类型的函数必须有不同的名字。OpenGL 已经把这类函数的命名约定标准化了。
>
> 诸如 `glUniform` 这样的函数带有一个后缀。后缀的第一部分是它接受的值的数量。在上面的例子中，`glUniform2f` 除了常规参数之外还接受 2 个值。`glUniform` 的基本参数只是 uniform 位置，所以 `glUniform2f` 接受的是 1 个 uniform 位置加 2 个值。
>
> 它所接受的值的类型由后缀的第二部分定义。可能出现的类型名及其对应类型如下：
>
> b
>
> 有符号字节
>
> ub
>
> 无符号字节
>
> s
>
> 有符号短整型
>
> us
>
> 无符号短整型
>
> i
>
> 有符号整型
>
> ui
>
> 无符号整型
>
> f
>
> 浮点型
>
> d
>
> 双精度浮点型
>
> 注意 OpenGL 为所有这些类型都提供了专门的 typedef。例如 `GLfloat` 就是 `float` 的 typedef。这对简单类型来说没什么特别的用处，但对整数类型而言，它们可能非常有用。
>
> 并不是所有接受多个值的函数都覆盖了所有可能的类型。例如 `glUniform` 只有 `i`、`ui` 和 `f` 三种变体（OpenGL 4.0 引入了 `d` 变体）。
>
> 此外还有向量形式，定义方式是在类型后面加一个 "v"。它们接受一个值数组，并用一个参数说明数组中有多少个元素。因此函数 `glUniform2fv` 接受 1 个 uniform 位置、1 个条目数量，以及一个具有相应条目数的数组。每个条目的大小是*两个浮点数*；条目数量不是浮点数或其他类型的数量。它等于值的总数除以每个条目中的值的数量。由于 "2f" 表示两个浮点数，这就是每个条目中值的数量。
>
> 有些 OpenGL 函数只接受类型而不带数字。这些函数接受固定数量的该类型参数，通常只有一个。

**向量数学**

你可能很好奇这几行是怎么工作的：

```cpp
vec4 totalOffset = vec4(offset.x, offset.y, 0.0, 0.0);
gl_Position = position + totalOffset;
```

这里看起来像函数的 `vec4` 其实是一个构造函数；它由 4 个浮点数创建一个 `vec4`。这么写是为了让加法更方便。

`position` 与 `totalOffset` 的相加是逐分量加法。它是下面这种写法的简写：

```cpp
gl_Position.x = position.x + totalOffset.x;
gl_Position.y = position.y + totalOffset.y;
gl_Position.z = position.z + totalOffset.z;
gl_Position.w = position.w + totalOffset.w;
```

GLSL 内置了大量向量数学运算。作用于向量时，这些数学运算都是逐分量进行的。不过，把不同维数的向量相加是违法的。所以你不能做 `vec2` + `vec4`。这就是为什么我们必须先把 `offset` 转换成 vec4 再做加法。

## 给着色器更多能力

我们不再需要手动变换顶点，这当然很好。但也许我们还能把更多东西搬到顶点着色器里。`ComputePositionOffsets` 能不能整体搬到顶点着色器中？

不过，不能。`glutGet(GL_ELAPSED_TIME)` 的调用搬不过去，因为 GLSL 代码不能直接调用 C/C++ 函数。但其余部分都可以搬过去。`VertCalcOffset.cpp` 做的正是这件事。

顶点程序位于 `data\calcOffset.vert`。

**示例：计算偏移量的顶点着色器**

```glsl
#version 330

layout(location = 0) in vec4 position;
uniform float loopDuration;
uniform float time;

void main()
{
    float timeScale = 3.14159f * 2.0f / loopDuration;
    
    float currTime = mod(time, loopDuration);
    vec4 totalOffset = vec4(
        cos(currTime * timeScale) * 0.5f,
        sin(currTime * timeScale) * 0.5f,
        0.0f,
        0.0f);
    
    gl_Position = position + totalOffset;
}
```

这个着色器接受两个 uniform：循环的持续时间以及已经经过的时间。

在这个着色器中，我们用到了若干标准 GLSL 函数，比如 `mod`、`cos` 和 `sin`。上一个教程里我们见过 `mix`。而这些还只是冰山一角；可用的标准 GLSL 函数有*很多*。

渲染代码与之前的渲染代码相当相似：

**示例：基于时间的渲染**

```cpp
void display()
{
    glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
    glClear(GL_COLOR_BUFFER_BIT);
    
    glUseProgram(theProgram);
    
    glUniform1f(elapsedTimeUniform, glutGet(GLUT_ELAPSED_TIME) / 1000.0f);
    
    glBindBuffer(GL_ARRAY_BUFFER, positionBufferObject);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);
    
    glDrawArrays(GL_TRIANGLES, 0, 3);
    
    glDisableVertexAttribArray(0);
    glUseProgram(0);
    
    glutSwapBuffers();
    glutPostRedisplay();
}
```

这一次，我们不需要任何代码去使用经过的时间；我们只需把它原封不动地传给着色器。

你可能想知道 `loopDuration` 这个 uniform 究竟是怎么被设置的。这是在我们的着色器初始化例程中完成的，而且只做一次：

**示例：设置循环持续时间**

```cpp
void InitializeProgram()
{
    std::vector<GLuint> shaderList;
    
    shaderList.push_back(Framework::LoadShader(GL_VERTEX_SHADER, "calcOffset.vert"));
    shaderList.push_back(Framework::LoadShader(GL_FRAGMENT_SHADER, "standard.frag"));
    
    theProgram = Framework::CreateProgram(shaderList);
    
    elapsedTimeUniform = glGetUniformLocation(theProgram, "time");
    
    GLint loopDurationUnf = glGetUniformLocation(theProgram, "loopDuration");
    glUseProgram(theProgram);
    glUniform1f(loopDurationUnf, 5.0f);
    glUseProgram(0);
}
```

我们照常使用 `glGetUniformLocation` 取得时间 uniform。对于循环持续时间，我们把它取到一个局部变量中。然后我们立即设置当前程序对象，把该 uniform 设为某个值，然后再取消当前程序对象。

程序对象和所有包含内部状态的对象一样，除非你显式修改，否则会一直保留自己的状态。因此 `loopDuration` 的值将永远是 5.0f；我们不需要每帧都去设置它。

## 多个着色器

让三角形动起来固然不错，但如果我们能在片元着色器里也做点基于时间的事情就更好了。片元着色器无法影响物体的位置，但它们可以控制其颜色。`FragChangeColor.cpp` 做的正是这件事。

本教程中的片元着色器从文件 `data\calcColor.frag` 加载：

**示例：基于时间的片元着色器**

```glsl
#version 330

out vec4 outputColor;

uniform float fragLoopDuration;
uniform float time;

const vec4 firstColor = vec4(1.0f, 1.0f, 1.0f, 1.0f);
const vec4 secondColor = vec4(0.0f, 1.0f, 0.0f, 1.0f);

void main()
{
    float currTime = mod(time, fragLoopDuration);
    float currLerp = currTime / fragLoopDuration;
    
    outputColor = mix(firstColor, secondColor, currLerp);
}
```

这个函数与顶点着色器中的周期循环类似（顶点着色器自我们上次见到它以来没有变化）。它不再用 sin/cos 函数计算圆周上的坐标，而是根据当前处在循环中的进度，在两种颜色之间做插值。当它位于循环起点时，三角形是 `firstColor`；当它位于循环终点时，三角形是 `secondColor`。

标准库函数 `mix` 在两个值之间执行线性插值。和许多 GLSL 标准函数一样，它可以接受向量参数，并对它们逐分量进行运算。因此两个参数的四个分量中的每一个，都会由第 3 个参数做线性插值。第 3 个参数在这里是 `currLerp`，取值在 0 到 1 之间。当它为 0 时，`mix` 的返回值就是第一个参数；当它为 1 时，返回值就是第二个参数。

程序初始化代码如下：

**示例：创建更多着色器**

```cpp
void InitializeProgram()
{
    std::vector<GLuint> shaderList;
    
    shaderList.push_back(Framework::LoadShader(GL_VERTEX_SHADER, "calcOffset.vert"));
    shaderList.push_back(Framework::LoadShader(GL_FRAGMENT_SHADER, "calcColor.frag"));
    
    theProgram = Framework::CreateProgram(shaderList);

    elapsedTimeUniform = glGetUniformLocation(theProgram, "time");
    
    GLint loopDurationUnf = glGetUniformLocation(theProgram, "loopDuration");
    GLint fragLoopDurUnf = glGetUniformLocation(theProgram, "fragLoopDuration");
    
    glUseProgram(theProgram);
    glUniform1f(loopDurationUnf, 5.0f);
    glUniform1f(fragLoopDurUnf, 10.0f);
    glUseProgram(0);
}
```

和之前一样，我们取得 `time` 和 `loopDuration` 的 uniform 位置，以及新增的 `fragLoopDuration` 的位置。然后为该程序设置两个循环持续时间。

你可能想知道顶点着色器和片元着色器的 `time` uniform 是怎么设置的？GLSL 编译模型有一个优点，它把顶点着色器和片元着色器链接成单个对象，因此同名同类型的 uniform 会被合并。所以 `time` 只有一个 uniform 位置，它同时指向两个着色器中的那个 uniform。

这样做的代价是，如果你在一个着色器中创建了一个与另一个着色器中某个 uniform 同名但*类型*不同的 uniform，OpenGL 会给出链接错误并无法生成程序。另外，还可能不小心把两个 uniform 链接成一个。在本教程中，片元着色器的循环持续时间不得不换一个名字，否则两个着色器就会共用同一个循环持续时间。

无论如何，正因为这个原因，渲染代码没有变化。时间 uniform 每帧都用 FreeGLUT 提供的经过时间来更新。

**着色器中的全局变量**

GLSL 中全局作用域的变量可以用特定的存储限定符来定义：`const`、`uniform`、`in` 和 `out`。`const` 值的作用与 C99 和 C++ 中一样：这个值就是不会变，没有例外。它必须带初始化器。没有限定符的变量则像 C/C++ 中预期的那样工作：它是一个可以被修改的全局值。GLSL 着色器可以调用函数，全局变量可以在函数之间共享。不过与 `in`、`out` 和 uniform 不同，非 const 变量和 `const` 变量*不会*在各个阶段之间共享。

## 关于顶点着色器的性能

这些教程都很简单，应该跑得足够快，但关注各种操作对性能的影响仍然很重要。本教程中我们给出了 3 种移动顶点数据的办法：在 CPU 上自己变换并上传到缓冲对象；在 CPU 上生成变换参数，让顶点着色器用它们来做变换；以及尽可能多地把东西放进顶点着色器，只让 CPU 提供最基础的参数。哪一种最好用？

这个问题不容易回答。不过几乎总是这样：CPU 上的变换会比在 GPU 上做要慢。唯一的例外是你需要在同一帧内反复做完全相同的变换。即便如此，也是在 GPU 上做一次变换、并存入缓冲对象供以后取用更为妥当。这称为变换反馈，我们会在后面的教程中讲。

至于另外两种方法究竟哪种更好，实际上取决于具体情形。以我们的例子来说：一种情况下，我们在 CPU 上计算偏移量并把它传给 GPU，由 GPU 把偏移量施加到每个顶点位置上。另一种情况下，我们只提供一个时间参数，而 GPU 必须为每个顶点算出*完全相同*的偏移量。这意味着顶点着色器做了大量工作，而结果全都是同一个数。

即便如此，这也不意味着它一定更慢。真正要紧的是改动数据的开销。更改一个 uniform 是要花时间的；由于许多显卡处理浮点运算的方式，改动一个向量 uniform 所需的时间通常并不比改动单个浮点数更多。问题在于：在顶点着色器中做更复杂的运算，其代价与这些运算需要被执行的频率相比，孰轻孰重。

我们使用的第二个顶点着色器，也就是自己计算偏移量的那个，做了大量复杂数学运算。正弦和余弦的求值速度并不算快，它们需要相当多的计算才能得出。而既然在一次渲染调用中偏移量对每个顶点都不变，从性能角度看，最好的做法是在 CPU 上计算偏移量，并把它作为 uniform 值传进去。

而通常情况下，渲染恰恰就是这么做的。顶点着色器拿到的变换值是在 CPU 上预先算好的。但这并不意味着这是唯一或最好的做法。在某些情况下，通过传给顶点着色器的参数化值来计算偏移量往往很有用。

这样做在顶点着色器的输入被抽象掉时最合适。也就是说，用户传的不是位置，而是一般化的信息，由着色器在某个时间点或其他参数下生成位置。这可以用于基于各种力的粒子系统；顶点着色器根据时间执行力函数，从而能够算出粒子在任意时刻的位置。

这样还有一个我们已经看到的好处。通过把高层信息传给着色器、让它去做复杂数学运算，你能影响的东西远不止一个简单的偏移量。片元着色器中的颜色动画只靠一个偏移量是不可能实现的。高层的参数化给了着色器极大的自由度。

## 本章小结

在本教程中，你学到了以下内容：

- 缓冲对象的内容可以用 `glBufferSubData` 函数做部分更新。这个函数执行的是等价于 `memcpy` 的操作。
- 着色器中的 uniform 变量，是由 GLSL 之外的代码设置的变量。它们只在渲染调用之间变化，因此在任何特定三角形的整个表面上都是均匀的。
- uniform 变量的值与程序对象一起存储。这一状态会一直保留，直到被显式修改。
- 在两个 GLSL 阶段中定义的、同名同类型的 uniform 变量被视为同一个 uniform。在程序对象中设置该 uniform，会同时改变两个阶段中的值。

### 进一步学习

有若干东西你可以动手试试，看看这些教程里会发生什么。

- 使用 `VertCalcOffset.cpp` 时，把它改成绘制两个沿圆周运动的三角形，其中一个比另一个领先半个 `loopDuration`。只需在 `glDrawArrays` 调用之后改变 uniform，然后再调用一次 `glDrawArrays` 即可。第二次设置之前，在时间里加上半个循环持续时间。
- 在 `FragChangeColor.cpp` 中，把它改成让片元程序在 `firstColor` 与 `secondColor` 之间来回反弹，而不是在循环末尾从 `secondColor` 突跳回 first。由 first 到 second 再回到 first 的过渡，都应该在单个 `fragLoopDuration` 时间区间内完成。顺带一提，GLSL 支持 `if` 语句以及 ?: 运算符。不过如果你想拿更多分，就不要用显式的条件语句；不妨用 sin 或 cos 函数来做这件事。
- 利用我们对 uniform 的认识，回到教程 2 的 FragPosition 教程。修改代码，让它接受一个描述窗口高度的 uniform，而不是使用硬编码值。修改 `reshape` 函数，绑定程序并用新的高度修改该 uniform。

### 值得关注的 OpenGL 函数

- **glBufferSubData**：该函数把内存从用户的内存地址复制进缓冲对象。它接受一个开始复制的字节偏移（相对于缓冲对象）以及要复制的字节数。

  当该函数把控制权交还给用户之后，你就可以立即释放自己拥有的那块内存。因此你可以分配并填充一块内存，调用这个函数，然后立刻释放那块内存，不会产生任何危险的副作用。OpenGL 不会保存该指针，之后也不会使用它。
- **glGetUniformLocation**：该函数从给定的程序对象中获取指定名称 uniform 的位置。如果该 uniform 不存在，或未被 GLSL 视为在使用中，该函数返回 -1，这不是有效的 uniform 位置。
- **glUniform***：把给定的 uniform 在当前使用的程序（由 `glUseProgram` 设置）设为给定值。这不只是一个函数，而是一整套接受不同类型的函数。

### 值得关注的 GLSL 函数

```cpp
mod(void);
```

`mod` 函数计算 `numerator` 对 `denominator` 的取模。可以把取模看作一种制造循环的方式；返回值会以循环的方式落在 [0,`denominator`) 区间内。在数学上，它被定义为 `numerator` - (`denominator` * `FLOOR`(`numerator`/`denominator`))，其中 `FLOOR` 把一个浮点值向下取整到最小的整数。

类型 `vec` 可以是 `float`，也可以是任意向量类型。所有参数必须是同一类型。如果使用了向量分母，那么取模会逐对应分量分别进行。该函数返回一个与 `numerator` 类型同样大小的向量。

```cpp
cos(void);
sin(void);
```

分别返回给定 `angle` 的三角函数[余弦或正弦](http://en.wikipedia.org/wiki/Sine#Sine.2C_cosine_and_tangent)。`angle` 的单位是弧度。如果 `angle` 是向量，返回的向量大小相同，并且是 `angle` 向量各分量的余弦或正弦。

## 术语表

- **uniform**：这是一类可以在 GLSL 着色器中定义的全局变量。它们表示在一次渲染操作过程中保持均匀（不变）的值。它们的值从着色器外部设置，并且无法在着色器内部更改。
