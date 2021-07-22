### [Xamarin] 解决WebView高度不能自适应问题

前几天在用WebView的时候遇到了高度无法自适应，只能通过手动设置高度临时解决了一下。但咱不能治标不治本呀，其实只需要在渲染完成后，获取真实高度并进行设置就行了，这里就记录一下解决方案。

首先自定义一个WebView
```c#
public class XamlWebView : WebView
{
}
```

在Android项目里
```c#
[assembly: ExportRenderer(typeof(XamlWebView), typeof(XamlWebViewRenderer))]
namespace XXX.Droid.UI
{
    public class XamlWebViewRenderer : WebViewRenderer
    {
        private static XamlWebView _xwebView = null;
        private Android.Webkit.WebView _webView;

        public XamlWebViewRenderer(Context context) : base(context)
        {

        }
        class XamWebViewClient : Android.Webkit.WebViewClient
        {
            public override async void OnPageFinished(Android.Webkit.WebView view, string url)
            {
                if (_xwebView != null)
                {
                    int i = 10;
                    while (view.ContentHeight == 0 && i-- > 0)
                        await System.Threading.Tasks.Task.Delay(50);// 这里的时间可以调整
                    _xwebView.HeightRequest = view.ContentHeight;
                }
            }
        }
        protected override void OnElementChanged(ElementChangedEventArgs<Xamarin.Forms.WebView> e)
        {
            base.OnElementChanged(e);
            _xwebView = e.NewElement as XamlWebView;
            _webView = Control;

            if (e.OldElement == null)
            {
                _webView.SetWebViewClient(new XamWebViewClient());
            }

        }
    }
}
```

在iOS项目里
```c#
[assembly: ExportRenderer(typeof(XamlWebView), typeof(XamWebViewRenderer))]
namespace XXX.iOS.UI
{
    public class XamWebViewRenderer : WkWebViewRenderer
    {
        protected override void OnElementChanged(VisualElementChangedEventArgs e)
        {
            base.OnElementChanged(e);
            this.WeakUIDelegate = new XamUIWebViewDelegate(this);
        }
    }

    public class XamUIWebViewDelegate : UIWebViewDelegate
    {
        XamWebViewRenderer webViewRenderer;

        public XamUIWebViewDelegate(XamWebViewRenderer _webViewRenderer = null)
        {
            webViewRenderer = _webViewRenderer ?? new XamWebViewRenderer();
        }

        public override async void LoadingFinished(UIWebView webView)
        {
            var wv = webViewRenderer.Element as XamlWebView;
            if (wv != null)
            {
                await System.Threading.Tasks.Task.Delay(50); // 这里的时间可以调整
                wv.HeightRequest = (double)webView.ScrollView.ContentSize.Height;
            }
        }
    }
}
```