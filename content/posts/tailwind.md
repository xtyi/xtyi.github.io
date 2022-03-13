---
title: Tailwind
date: 2022-03-13T16:42:19+08:00
draft: false
tags:
  - web
---



# 传统 CSS 框架的缺陷

从学习前端技术以来，我都觉得 CSS 是特别难处理的东西，以前写页面很随意，直接使用 Boostrap、Bulma 等 CSS 框架

如果用前端框架开发，那就使用对应的 BootstrapVue、ElementUI 等组件库

这些框架都定义了一套设计语言(风格)，提供许多的组件，基本上能做一个看起来不错的网站

如果你需要搞一个后台管理界面，或者这些框架的设计语言正好符合你的审美，那这些框架还是不错的

但是如果要做一个面向用户的，可能维护很久的 web app，一般要有专门的设计师来设计 UI，开发者要根据设计稿还原

这种情况下，使用这些框架就很麻烦了，首先这些框架自带样式，这些样式很可能和设计稿的样式冲突，你还要覆盖掉这些框架的样式，十分麻烦

那么只能手写 CSS 了吗？

# Tailwind

Tailwind 是近些年流行起来的一个 CSS 框架，它和原先的 CSS 框架(例如 Bootstrap)的不同在于：Tailwind 提供的 class 都是描述一个原子样式的，而不是描述一个组件

Bootstrap 的大部分 class 都是像 `.btn` `.btn-primary` 这样描述一个组件

```html
<button type="button" class="btn btn-primary">Primary</button>
```

嗯，写起来很方便，但是样式就是这样了，虽然也可以自定义主题，但作用有限，想要完全掌控页面的样式，还是不太现实的

Tailwind 呢？

```html
<button class="text-white bg-[#0d6efd] border-[1px] border-[#0d6efd] hover:bg-[#0b5ed7] hover:border-[#0a58ca] px-2 py-1 rounded-md font-normal">Primary</button>
```

确实更灵活了，但是，这不就是内联样式吗？！

而且学习 CSS 的时候一定被教导过，不要写这样的类

```css
.text-white {
	color: white;
}
```

一个类应该描述一个组件，而且要好好命名，遵循 BEM 规范等等，例如

```html
<div class="card">
  <div class="card-header"></div>
  <div class="card-body"></div>
  <div class="card-footer">
  	<button class="btn btn-primary">OK</button>
    <button class="btn btn-secondary">Cancel</button>
  </div>
</div>
```

确实，过去有点追求的前端工程师都是这么做的，包括 Bootstrap，ElementUI 等框架的也是这种设计

拥护这种方法的说辞无非就是更语义化，不容易样式冲突，总之就是更容易维护

在前端刀耕火种的时代，这种说法有点道理

# 前端组件化

但是随着前端开发组件化，我们完全用 Vue 组件，React 组件来抽象页面中独立的部分，class 语义化的作用进一步被削弱，class 只剩下了用于附加 CSS 样式的功能

再认真思考，软件设计的精髓在于分辨软件中容易改变的部分和不容易改变的部分，采用不同的处理方法

而 CSS 样式就是容易改变的部分，既然 class 的功能只是附加一些容易改变的样式，那么花费精力去抽象这些 CSS，想一个好的 class 名字，值得吗？

如今，class 的命名极大的阻碍了开发效率，并且得不到任何好处

# 关注点分离

React 发布的时候，很多人质疑 React 的 JSX 语法：这不就是 PHP 吗？

起初大家按照语言来分离关注点，这是简单的，符合直觉的

但是时间证明了一切，把 HTML，CSS，JS 分开写是错误的！

控制 UI 的逻辑就应该和 UI 一起写

我们应该按照业务(组件)来分离关注点

所以，样式难道不应该就和 HTML 写在一起吗，不然即使在同一个文件中，一会跳到上面写 HTML，一会滑倒下面写 CSS，绝对是阻碍开发效率的

示例：Vue 的单文件组件

```vue
<template>

</template>

// ....

<style>

</style>
```

Tailwind 让你不用再写 CSS，tailwind 提供的一个 class 就是一个原子样式

只需要在标签上写 class，就可以完成这个标签的样式

和 React 一样，Tailwind 现在也被很多人质疑，但时间会证明一切



# Tailwind Features

实际上 Bootstrap 等框架也提供了一些这样的 class，被其称作 helpers 或是 utilities，但是做到这种程度远远不够

Tailwind 除了提供详尽的 class 外，还支持以下功能：

1. 任意值

   可以使用预定义的字体大小，字体颜色，padding，边框圆角等等

   ```html
   <div class="text-md text-blue-500 px-2 rounded-xl">Blue</div>
   ```

   也可以任意指定

   ```html
   <div class="text-[17px] bg-[#0d6efd] px-[5px] rounded-[20px]">Blue</div>
   ```

2. 状态

   可以很方便的设置元素处于某种状态时的样式

   ```html
   <button class="hover:bg-[#0d6efd]"></button>
   ```

3. 编译

   通过编译生成最后的 CSS

# Tailwind 与前端组件库

目前的前端组件库还是将传统的 CSS 框架组件化，每一个组件库都有一套自己的设计语言，例如

BootstrapVue -> Bootstrap

Buefy -> Bulma

Ant Design Vue -> Ant Design

Quasar -> Material

有没有框架能和 Tailwind 结合呢，其实也就是提供一些常用的不太好写的组件，例如数据展示的 Table，一些表单相关的组件

这些组件把功能实现好，只实现基础的样式，并且设置 props 可以把 class 传递给组件内部一些关键的元素上

[VueTailwind](https://www.vue-tailwind.com/) 和 [HeadlessUI](https://headlessui.dev/) 就是这样的框架，不过目前还不成熟























































