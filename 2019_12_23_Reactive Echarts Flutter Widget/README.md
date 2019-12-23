> Introducing the development of a reactive Echarts Flutter Widget: [flutter_echarts](https://github.com/entronad/flutter_echarts) .
>
> [repository](https://github.com/entronad/flutter_echarts) 
>
> [pub.dev](https://pub.dev/packages/flutter_echarts) 

As the fast developing of Flutter, it has been applied to more and more massive projects, and complex data visualization charts has been an important requirement. Although Flutter has powerful classes like Painter or Canvas to do the painting work, unfortunately, there is sill no killer data visualization library in the Flutter ecosystem.

Early in this year, the Flutter development group has published an official inline WebView widget: [webview_flutter](https://pub.dev/packages/webview_flutter) . It is based on the new  Platform View, which makes it possible to embed web contents seamlessly in Flutter, just like other widgets. Thus we can import those mature web data visualization libraries into our Flutter apps.

When talks about mature, powerful and easy-to-use data visualization libraries, [Echarts](https://www.echartsjs.com/zh/index.html) is defiantly a good choice. I would repeat no more about it's advantages here. If we can add Echarts to our Flutter apps, we can not only implement abundant chart types it supports, but also reuse the ready-made chart codes of the web to reduce the workload.

So we encapsulated a Flutter widget: [flutter_echarts](https://github.com/entronad/flutter_echarts) , which taking into account both extensibility and ease to use

