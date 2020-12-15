# Azure Functions カスタムメトリックハンズオン
　  
- 関数を作成し、Azure の App Service からメトリックを取得し Azure Monitor のカスタム メトリックとして表示します。
- ロールを割り当てて Azure リソースへのアクセスを付与するために使用するアクセス制御 (IAM) の基本が体験できます。
- Azure のリソースを Functions を使って操作する方法を体験することがメインになっており、C# のコードを書くことがメインではありません。 ※あくまでも簡単に体験していただくものです。ベストプラクティスなどをご提案するものではありません。
　  
 
# 関数アプリ #
　  
## 関数アプリのリソース作成  ##
　  
![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f001.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f002.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f003.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f004.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f005.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f006.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f007.png?raw=true)
  

## TimerTrigger 関数を作成  ##
　  
![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f008.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f009.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f010.png?raw=true)
  

## リソースへのマネージド ID の割り当て ##
　  
![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f011.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f012.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f013.png?raw=true)
  

## リソースに Azure ロールの割り当て ##
　  
![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f014.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f015.png?raw=true)
  

![](https://github.com/TomohiroSuzuki128/AzureFunctionsCustomMetricsHandsOn/blob/main/images/f016.png?raw=true)
  





**function.proj**

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>netstandard2.0</TargetFramework>
    </PropertyGroup>
    <ItemGroup>
        <PackageReference Include="System.Json" Version="4.7.1" />
        <PackageReference Include="System.Text.Json" Version="5.0.0" />
        <PackageReference Include="System.Net.Http" Version="4.3.4" />
        <PackageReference Include="System.Security.Cryptography.ProtectedData" Version="4.7.0" />
        <PackageReference Include="Microsoft.Azure.Devices" Version="1.20.1" />
        <PackageReference Include="Microsoft.Azure.Management.Fluent" Version="1.33.0" />
        <PackageReference Include="Microsoft.Azure.Services.AppAuthentication" Version="1.4.0" />
        <PackageReference Include="Microsoft.Azure.WebJobs.Extensions.Http" Version="3.0.2" />
        <PackageReference Include="Microsoft.Identity.Client" Version="4.13.0" />
    </ItemGroup>
</Project>
```
　  
　  

**run.csx**

```csharp
<?xml version="1.0" encoding="utf-8" ?>
using System;
using System.Linq;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;
using Microsoft.Azure.Services.AppAuthentication;
using Microsoft.Identity.Client;

private static HttpClient httpClient = new HttpClient();

private static string subscriptionID = "<your subscription ID>";
private static string tenantID = "<your tenant ID>";
private static string clientID = "<your client ID>";

private static string secret = "<your secret>";
private static string resourceGroupName = "CustomMetricHandsOn";
private static string webAppName = "<your web app name>";

public static async void Run(TimerInfo myTimer, ILogger log)
{
    log.LogInformation($"TimerTriggerCustomMetric function executed at: {DateTime.Now}");

    var currentValue = string.Empty;
    var limit = string.Empty;

    log.LogInformation($"--- 1 Get an access token for App Service ---");

    IConfidentialClientApplication app = ConfidentialClientApplicationBuilder.Create(clientID)
        .WithClientSecret(secret)
        .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantID}"))
        .Build();

    var scopes = new string[] {"https://management.azure.com/.default"};
    var result = await app.AcquireTokenForClient(scopes).ExecuteAsync();

    //log.LogInformation($"Bearer : {result.AccessToken}");

    log.LogInformation($"--- 2 Get metrics from App Service ---");

    var requestUriAppService = $"https://management.azure.com/subscriptions/{subscriptionID}/resourceGroups/{resourceGroupName}/providers/Microsoft.Web/sites/{webAppName}/usages?api-version=2019-08-01&filter=(name.value eq 'FileSystemStorage')";
    httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", result.AccessToken);

    var getRequest = new HttpRequestMessage
    {
        Method = HttpMethod.Get,
        RequestUri = new Uri(requestUriAppService)
    };

    var getResponse = await httpClient.SendAsync(getRequest);
    var getResponseStr = await getResponse.Content.ReadAsStringAsync(); 

    log.LogInformation(getResponseStr);

    var metrics = JsonDocument.Parse(getResponseStr);
    var metric = metrics.RootElement.GetProperty("value").EnumerateArray().Last();
    currentValue = metric.GetProperty("currentValue").GetInt32().ToString();
    limit = metric.GetProperty("limit").GetInt64().ToString();

    log.LogInformation($"currentValue = {currentValue}");
    log.LogInformation($"limit = {limit}");

    log.LogInformation($"--- 3 Create Custom Metric JSON ---");

    var nowDate = DateTime.Now.ToString("s");
    var json = "{" +
            $"\"time\":\"{nowDate}\"," +
            "\"data\":{" +
            "\"baseData\":{" +
            "\"metric\":\"File System Storage Usage Custom\"," +
            "\"namespace\":\"My Custom Metrics\"," +
            "\"dimNames\":[" +
            "\"Usage\"]," +
            "\"series\":[" +
            "{" +
            "\"dimValues\":[" +
            "\"File System Storage\"" +
            "]," +
            $"\"min\":0," +
            $"\"max\":{limit}," +
            $"\"sum\":{currentValue}," +
            $"\"count\":1" +
            "}" +
            "]" +
            "}" +
            "}" +
            "}";

    //log.LogInformation($"JSON - {json}");

    log.LogInformation($"--- 4 Post the metric to Monitor ---");

    var azureServiceTokenProvider = new AzureServiceTokenProvider();
    var token = await azureServiceTokenProvider.GetAccessTokenAsync("https://monitoring.azure.com/");

    //log.LogInformation($"Bearer : {token}");

    var requestUriMonitor = $"https://japaneast.monitoring.azure.com/subscriptions/{subscriptionID}/resourceGroups/{resourceGroupName}/providers/Microsoft.Web/sites/{webAppName}/metrics";
    httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);

    var content = new StringContent(json, Encoding.UTF8, @"application/json");
    var postResponse = await httpClient.PostAsync(requestUriMonitor, content);
    var postResponseStr = await postResponse.Content.ReadAsStringAsync();

    if(string.IsNullOrEmpty(postResponseStr))
        log.LogInformation($"--- Finished !! success !! ---");

}
```
　  
これで、ハンスオンは終了です。お疲れ様でした。
　  
　  