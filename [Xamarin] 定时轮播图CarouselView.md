### [Xamarin] 定时轮播图CarouselView

自带的CarouselView虽然可以轮播，但只能手动去触发，要实现轮播图就需要定制CarouselView  

```c#
public class XamlCarouselView : CarouselView
    {
        public static readonly BindableProperty IntervalProperty =
        BindableProperty.Create(nameof(Interval), typeof(int), typeof(XamlCarouselView), 0,
            defaultBindingMode: BindingMode.TwoWay,
            propertyChanged: IntervalPropertyChanged);

        public XamlCarouselView() : base()
        {
        }

        /// <summary>
        /// 切换图片时间间隔，单位：秒
        /// </summary>
        public int Interval
        {
            get { return (int)GetValue(IntervalProperty); }
            set { SetValue(IntervalProperty, value); }
        }

        private Timer _timer;

        private void OnTime(int interval)
        {
            if (interval <= 0 && _timer != null)
            {
                _timer.Dispose();
                _timer = null;
                return;
            }

            if (_timer == null)
                _timer = new Timer(Carousel, null, TimeSpan.Zero, TimeSpan.FromSeconds(interval));
            else
                _timer.Change(TimeSpan.Zero, TimeSpan.FromSeconds(interval));
        }

        private static void IntervalPropertyChanged(BindableObject bindable, object oldValue, object newValue)
        {
            var control = (XamlCarouselView)bindable;
            if (newValue != null)
                control.OnTime((int)newValue);
        }

        private void Carousel(object state)
        {
            var items = this.ItemsSource;
            if (items == null)
                return;

            var lenght = 0;
            foreach (var item in items)
                lenght++;

            if (lenght <= 0)
                return;

            var position = (this.Position + 1) % lenght;
            this.ScrollTo(position);
        }
    }
```

在xaml中使用
```xaml
<StackLayout>
    <local:XamlCarouselView ItemsSource="{Binding AdvImgls}" Interval="5">
        <local:XamlCarouselView.ItemTemplate>
            <DataTemplate>
                <Image Source="{Binding url}" />
            </DataTemplate>
        </local:XamlCarouselView.ItemTemplate>
    </local:XamlCarouselView>
</StackLayout>
```

顺便吐槽一下，Xamarin自带的控件又少功能还弱，emmmmm......
