# CoreUI

coreui 是一个基于 Bootstrap 的 UI 库, 完全兼容 Bootstrap 的语法.

## Modal

Modal 是 Boostrap 中的一种控件, 也就是俗称的弹窗.  Modal 使用 HTML, CSS 和 JavaScript 制作而成. 它们处于页面中所有其它的元素之前, 并且取代了 <body> 的滚动条, 这样 modal 的内容就可以滚动.

点击 modal 上的 "backdrop" 按钮, 就会自动关闭 modal.

Bootstrap 只支持一次显示一个 modal 窗口. 不支持嵌套 modals, 是因为我们相信这样会造成很差的用户体验.

Modal 使用 `position: fixed`, 有时这会影响到它的渲染. 尽可能地让你的 modal HTML 处于顶层的位置, 以避免对其它元素潜在的影响. 将 modal 嵌套在另一个 fixed 的元素内可能会有问题.

另外, 由于 `position: fixed`, modal 在移动端的使用可能会有问题.

由于 HTML5 的机制, `autofocus` 属性在 modal 内无效. 为了达到相同效果, 可以使用以下 JavaScript:

```js
$('#myModal').on('shown.bs.modal', function () {
  $('#myInput').trigger('focus')
})
```