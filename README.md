# X5-ActionMode

腾讯X5内核实现长按弹出自定义菜单方法
https://blog.csdn.net/TLuffy/article/details/109174837#commentBox

非常感谢博主的文章和帮助


上面的解决方案是屏蔽了X5自身的ActionMode，但是自定义的ActionMode是通过js的方式去解决的，那么原生有没有办法解决呢，答案是有的。

下面是代码
```
    private WebView mWebView;

    boolean isInActionMode = false;
    ActionMode actionMode;
    View actionView;

    IX5WebViewExtension x5WebViewExtension = webView.getX5WebViewExtension();
    if (x5WebViewExtension != null) {
        x5WebViewExtension.setWebViewClientExtension(new ProxyWebViewClientExtensionX());
        x5WebViewExtension.setSelectListener(new ISelectionInterfaceX());
    }

    // 腾讯X5内核实现长按弹出自定义菜单方法
    // https://blog.csdn.net/TLuffy/article/details/109174837
    // 文章是屏蔽了长按弹框，通过js实现自定义弹框，但我们是通过原生实现
    private class ProxyWebViewClientExtensionX extends ProxyWebViewClientExtension {
        @Override
        public boolean onShowLongClickPopupMenu() {
            postDelayed(() -> mWebView.getX5WebViewExtension().enterSelectionMode(false), 30);
            return true;
        }
    }

    // x5相关api介绍
    // https://x5.tencent.com/docs/tbsapi.html
    private class ISelectionInterfaceX implements ISelectionInterface {

        @Override
        public void onSelectionChange(Rect rect, Rect rect1, int i, int i1, short i2) {
            System.out.println();
        }

        @Override
        public void onSelectionBegin(Rect rect, Rect rect1, int i, int i1, short i2) {
            System.out.println();
        }

        @Override
        public void onSelectionBeginFailed(int i, int i1) {
            System.out.println();
        }

        @Override
        public void onSelectionDone(Rect rect, boolean b) {
            System.out.println();
        }

        @Override
        public void hideSelectionView() {
            System.out.println();
            if (actionView != null) {
                mWebView.removeViewInLayout(actionView);
                actionView = null;
            }
            if (actionMode != null) {
                actionMode.finish();
                actionMode = null;
            }
        }

        @Override
        public void onSelectCancel() {
            System.out.println();
        }

        @Override
        public void updateHelperWidget(Rect rect, Rect rect1) {
            System.out.println();
            post(() -> {

                ActionMode.Callback callback = new ActionMode.Callback() {
                    @Override
                    public boolean onCreateActionMode(ActionMode mode, Menu menu) {
                        // 引入新的menu
                        mode.getMenuInflater().inflate(R.menu.menu_imgtxt_detail, menu);
                        return true;
                    }

                    @Override
                    public boolean onPrepareActionMode(ActionMode mode, Menu menu) {
                        return false;
                    }

                    @Override
                    public boolean onActionItemClicked(ActionMode mode,
                                                       MenuItem item) {
                        IX5WebViewExtension webViewExtension = mWebView.getX5WebViewExtension();
                        String selectionText = "";
                        if (webViewExtension != null) {
                            selectionText = webViewExtension.getSelectionText();
                        }
                        boolean leaveSelection = true;
                        switch (item.getItemId()) {
                            case R.id.menu_copy:
                                ClipboardUtils.copyText(selectionText);
                                ToastUtils.showShort(R.string.copy_already);
                                break;
                            case R.id.menu_dict:
                                leaveSelection = false;
                                break;
                            case R.id.menu_search:
                                RouterUtils.switchToSearchActivity("", selectionText);
                                break;
                        }
                        boolean finalLeaveSelection = leaveSelection;
                        postDelayed(() -> {
                            if (finalLeaveSelection && webViewExtension != null) {
                                webViewExtension.leaveSelectionMode();
                            }
                        }, 30);
                        return true;
                    }

                    @Override
                    public void onDestroyActionMode(ActionMode mode) {
                        isInActionMode = false;
                    }
                };

                Context context = getContext();
                if (context instanceof BaseActivity) {
                    // 生成actionView，并通过它启动ActionMode，解决定位问题
                    // actionView的大小和位置，大致和Selection相同，直接将其add到WebView中
                    if (actionView != null) {
                        mWebView.removeViewInLayout(actionView);
                    }

                    actionView = new View(context);
                    actionView.setBackgroundColor(0x33FF00FF);// 方便调试
                    int width = rect1.right - rect.left;
                    int height = rect1.bottom - rect.top;
                    LayoutParams lp = new LayoutParams(width <= 0 ? 10 : width, height <= 0 ? 10 : height);
                    lp.leftMargin = rect.left;
                    lp.topMargin = rect.top;
                    mWebView.addView(actionView, lp);

                    // 需要延迟startActionMode，给布局actionView的时间
                    actionView.post(() -> {
                        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                            actionMode = actionView.startActionMode(callback, ActionMode.TYPE_FLOATING);
                        } else {
                            actionMode = actionView.startActionMode(callback);
                        }
                    });
                }
                isInActionMode = true;
            });
        }

        @Override
        public void setText(String s, boolean b) {
            System.out.println();
        }

        @Override
        public String getText() {
            System.out.println();
            return null;
        }

        @Override
        public void onRetrieveFingerSearchContextResponse(String s, String s1, int i) {
            System.out.println();
        }
    }
```

这里面用了 mWebView.addView(actionView, lp)和mWebView.removeViewInLayout(actionView)去add和remove对应的View。
为什么不是addView和removeView呢，因为x5的webview重写了addView(View child)和removeView(View child)方法，
但是没重写addView(View child, LayoutParams params)，所以导致
通过addView(View child, LayoutParams params)方法加入的view无法通过removeView(View child)删除，所以改用了removeViewInLayout(View child)去删除。

