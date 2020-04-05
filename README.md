# CoreProfile.Web 實現跨應用程式性能調整及監控

## 適用情境:
- 跨應用程式性能調整及監控

## NuGet套件:
- coreprofiler
- CoreProfiler.Web

## 環境要求:
- WebService應用程式須安裝CoreProfile或NanoProfile，且DrillDown功能須設定為開啟，允許從外部應用程序向下鑽取子請求。

- Net.Core專案 Startup.cs 必須加上 app.UseCoreProfiler(true)。 (參數為true表示開啟跨應用程式，從外部應用程序向下取子請求drillDown功能)


```
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseCoreProfiler(true);

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseHttpsRedirection();

            app.UseRouting();

            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
```

## 實作內容:

------------
### 方法一 :  ProfilingSession.Current.WebTimingAsync

```
            // WebTimingAsync() profiles the wrapped action as a web request timing
            var url = this.Request.Scheme + "://" + this.Request.Host + "/home/child";
            await ProfilingSession.Current.WebTimingAsync(url, async (correlationId) =>
            {
                using (var httpClient = new HttpClient())
                {
                    httpClient.DefaultRequestHeaders.Add(CoreProfilerMiddleware.XCorrelationId, correlationId);

                    var uri = new Uri(url);
                    var result = await httpClient.GetStringAsync(uri);
                }
            });
```


* HttpClient發送前，必需要在HttpHeader加上CorrelationId，才能取得外部應用程式CoreProfiler或NanoProfiler的監控紀錄。

* HttpClient都需要加上以上的程式碼，會過於繁重，個人比較推薦使用第二種方法，亦可使用設定的方式進行移除或裝載功能。

------------
### 方法二 : HttpClientFactory 實作 DelegatingHandler

~/Infrastructure.DelegatingHandlers.CoreProfilerDelegatingHandler.cs

```
/// <summary>
    /// CoreProfilerDelegatingHandler
    /// </summary>
    /// <seealso cref="System.Net.Http.DelegatingHandler" />
    public class CoreProfilerDelegatingHandler : DelegatingHandler
    {
        /// <summary>
        /// Sends an HTTP request to the inner handler to send to the server as an asynchronous operation.
        /// </summary>
        /// <param name="request">The HTTP request message to send to the server.</param>
        /// <param name="cancellationToken">A cancellation token to cancel operation.</param>
        /// <returns>
        /// The task object representing the asynchronous operation.
        /// </returns>
        protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        {
            if (ProfilingSession.Current == null)
            {
                return await base.SendAsync(request, cancellationToken);
            }

            string url = request.RequestUri.AbsoluteUri;

            var webTiming = new WebTiming
            (
                profiler: ProfilingSession.Current.Profiler, 
                url: url
            );
            try
            {
                request.Headers.Add
                (
                    CoreProfilerMiddleware.XCorrelationId,
                    webTiming.CorrelationId
                );

                return await base.SendAsync(request, cancellationToken);
            }
            finally
            {
                webTiming.Stop();
            }
        }
    }
```
~/Startup.cs
```
  public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<HttpClient>();
            services.AddEvertrustUrls(Configuration);

            services.AddSingleton<IWebServiceUrlHelper, WebServiceUrlHelper>()
                    .AddTransient<ICountyRepository, CountyRepository>();

            services.AddTransient<CoreProfilerDelegatingHandler>()
                    .AddHttpClient(HttpClientNames.CoreProfiler)
                    .AddHttpMessageHandler<CoreProfilerDelegatingHandler>();

            services.AddControllers();
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseCoreProfiler(true);

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseHttpsRedirection();

            app.UseRouting();

            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
```

~/Infrastructure.Repository.CountyRepository.cs

```
        /// <summary>
        /// 取得所有縣市
        /// </summary>
        /// <returns></returns>
        public async ValueTask<IList<CountyEntity>> GetAllAsync()
        {
            var url = $"{_webServiceUrlHelper.HFWS_YCHF_Common}/api/geography/county";

            var httpClient = this._httpClientFactory.CreateClient(HttpClientNames.CoreProfiler);

            var response = await httpClient.GetAsync(url)
                                           .ConfigureAwait(false);

            response.EnsureSuccessStatusCode();

            await using var responseStream = await response.Content.ReadAsStreamAsync();

            var successResult = await JsonSerializer.DeserializeAsync<SuccessResultModel<IList<CountyEntity>>>(responseStream);

            return successResult.Data;
        }
```

* 須使用HttpClientFactory已註冊過使用的name，已經註冊的HttpClient已加入CoreProfilerDelegatingHandler。


##  參考資料:

HttpClientFactory:
* https://docs.microsoft.com/zh-tw/aspnet/core/fundamentals/http-requests?view=aspnetcore-3.1

CoreProfiler:
* https://github.com/teddymacn/CoreProfiler
* https://github.com/teddymacn/cross-app-profiling-demo
* https://www.itread01.com/articles/1475204636.html
