# Geofences

![A sketchnote overview of this lesson](../../../sketchnotes/lesson-14.jpg)

> Sketchnote by [Nitya Narasimhan](https://github.com/nitya). Click the image for a larger version.

This video gives an overview of geofences and how to use them in Azure Maps, topics that will be covered in this lesson:

[![Geofencing with Azure Maps from the Microsoft Developer IoT show](https://img.youtube.com/vi/nsrgYhaYNVY/0.jpg)](https://www.youtube.com/watch?v=nsrgYhaYNVY)

> 🎥 Click the image above to watch a video

## Pre-lecture quiz

[Pre-lecture quiz](https://black-meadow-040d15503.1.azurestaticapps.net/quiz/27)

## Introduction

In the last 3 lessons, you have used IoT to locate the trucks carrying your produce from your farm to a processing hub. You've captured GPS data, sent it to the cloud to store, and visualized it on a map. The next step in increasing the efficiency of your supply chain is to get an alert when a truck is about to arrive at the processing hub, so that the crew needed to unload can be ready with forklifts and other equipment as soon as the vehicle arrives. This way they can unload quickly, and you are not paying for a truck and driver to wait.

In this lesson you will learn about geofences - defined geospatial regions such as an area within a 2km minute drive of a processing hub, and how to test if GPS coordinates are inside or outside a geofence, so you can see if your GPS sensor has arrived or left an area.

In this lesson we'll cover:

- [What are geofences](#what-are-geofences)
- [Define a geofence](#defining-a-geofence)
- [Test points against a geofence](#testing-points-against-a-geofence)
- [Use geofences from serverless code](#use-geofences-from-serverless-code)

> 🗑 This is the last lesson in this project, so after completing this lesson and the assignment, don't forget to clean up your cloud services. You will need the services to complete the assignment, so make sure to complete that first.
>
> Refer to [the clean up your project guide](../../../clean-up.md) if necessary for instructions on how to do this.

## What are Geofences

A geofence is a virtual perimeter for a real-world geographic region. Geofences can be circles defined as a point and a radius (for example a circle 100m wide around a building), or a polygon covering an area such as a school zone, city limits, or university or office campus.

![Some geofence examples showing a circular geofence around the Microsoft company store, and a polygon geofence around the Microsoft west campus](../../../images/geofence-examples.png)

> 💁 You may have already used geofences without knowing. If you've set a reminder using the iOS reminders app or Google Keep based off a location, you have used a geofence. These apps will set up a geofence based off the location given and alert you when your phone enters the geofence.

There are many reasons why you would want to know that a vehicle is inside or outside a geofence:

- Preparation for unloading - getting a notification that a vehicle has arrived on-site allows a crew to be prepared to unload the vehicle, reducing vehicle waiting time. This can allow a driver to make more deliveries in a day with less waiting time.
- Tax compliance - some countries, such as New Zealand, charge road taxes for diesel vehicles based on the vehicle weight when driving on public roads only. Using geofences allows you to track the mileage driven on public roads as opposed to private roads on sites such as farms or logging areas.
- Monitoring theft - if a vehicle should only remain in a certain area such as on a farm, and it leaves the geofence, it might have been stolen.
- Location compliance - some parts of a work site, farm or factory may be off-limits to certain vehicles, such as keeping vehicles that carry artificial fertilizers and pesticides away from fields growing organic produce. If a geofence is entered, then a vehicle is outside of compliance and the driver can be notified.

✅ Can you think of other uses for geofences?

Azure Maps, the service you used in the last lesson to visualize GPS data, allows you to define geofences, then test to see if a point is inside or outside of the geofence.

## 지오펜스(Geofence) 정의

지오펜스는 이전 학습에서 지도에 추가한 지점과 동일하게 GeoJSON을 사용하여 정의됩니다. 이 경우, `FeatureCollection` 의 `Point` 값 대신, `FeatureCollection` 이 `Polygon`을 포함하게 됩니다.

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [-122.13393688201903, 47.63829579223815],
            [-122.13389128446579, 47.63782047131512],
            [-122.13240802288054, 47.63783312249837],
            [-122.13238388299942, 47.63829037035086],
            [-122.13393688201903, 47.63829579223815]
          ]
        ]
      },
      "properties": {
        "geometryId": "1"
      }
    }
  ]
}
```

다각형의 각 점은 경도, 위도의 배열 쌍으로 정의되며, 이러한 점들은 `coordinates` 로 설정된 배열에 있습니다. 지난 수업의 `Point`에서, `coordinates`는 위도와 경도 2 개의 값을 포함하는 배열이었습니다. `Polygon`은 위도와 경도 배열을 포함하는 배열입니다.

> 💁 기억하세요, GeoJSON 은 좌표에 `latitude, longitude`가 아닌 `longitude, latitude` 를 사용합니다.

다각형 좌표 배열에는 항상 다각형의 점 수보다 1개 더 많은 항목이 있으며, 마지막 항목은 첫 번째 항목과 동일한 것으로 다각형을 닫습니다. 예를 들어 직사각형의 경우 5개의 점이 있습니다.

![A rectangle with coordinates](../../../../images/polygon-points.png)

다각형 좌표는 왼쪽 상단의 47, -122에서 시작해 오른쪽으로 47, -121 이동한 다음, 아래로 46, -121 이동하고, 오른쪽으로 46, -122 이동한 다음 다시 시작점인 47, -122로 돌아옵니다. 이렇게 하면 다각형에 왼쪽 위, 오른쪽 위, 오른쪽 아래, 왼쪽 아래, 왼쪽 위의 5개의 점이 제공되어 다각형을 닫습니다.

✅ 집이나 학교 주변의 GeoJSON 다각형을 만들어 보세요. [GeoJSON.io](https://geojson.io/) 같은 도구를 사용하세요.

### 작업 - 지오펜스 정의

Azure Maps에서 지오펜스를 사용하려면, 먼저 Azure Maps 계정에 업로드해야 합니다.업로드하면 지오펜스에 대한 지점을 테스트할 수 있는 고유 ID를 받게 됩니다. 지오펜스를 Azure Maps에 업로드하려면, 지도 웹 API를 사용해야 합니다. [curl](https://curl.se) 이라는 도구를 사용해 Azure Maps web API를 호출할 수 있습니다.

> 🎓 Curl 은 웹 끝점에 대한 요청을 만드는 명령어 도구입니다.

1. Linux, macOS, 또는 최신 버전의 Windows 10 를 사용하는 경우 curl 이 이미 설치되어 있을 수 있습니다. 터미널 또는 명령어에서 다음을 실행하여 확인합니다:

   ```sh
   curl --version
   ```

   curl의 버전 정보가 표시되지 않으면, [curl 다운로드 페이지](https://curl.se/download.html)에서 설치해야 합니다.

   > 💁 Postman를 사용한 경헙이 있다면, 원하는 경우 이를 대신 사용할 수 있습니다.

2. 다각형이 포함된 GeoJSON 파일을 생성합니다. GPS 센서를 사용하여 이를 테스트할 것이므로, 현재의 위치 주변에 다각형을 만듭니다. 위에 제공된 GeoJSON 예제를 편집하여 수동으로 만들거나, [GeoJSON.io](https://geojson.io/) 같은 도구를 사용할 수 있습니다.

   GeoJSON은 `Polygon` 타입의 `geometry`의 `Feature`를 포함한 `FeatureCollection`을 가지고 있습니다.

   **반드시** `geometry` 요소와 동일한 레벨의 `properties` 요소를 추가해야 합니다. 이는 `geometryId`를 포함하고 있습니다:

   ```json
   "properties": {
       "geometryId": "1"
   }
   ```

   [GeoJSON.io](https://geojson.io/)를 사용하는 경우, JSON 파일을 다운로드 한 후, 또는 JSON 편집기에서 비어있는 `properties` 요소에 수동으로 이 항목을 추가해야 합니다.

   이 `geometryId` 는 이 파일에서 고유해야 합니다. You can upload multiple geofences as multiple `Features` in the `FeatureCollection` in the same GeoJSON file, as long as each one has a different `geometryId`. 다각형은 다른 파일에서 다른 시간대에 업로드 된 경우, 동일한 `geometryId`를 가질 수 있습니다.

3. 이 파일을 `geofence.json`로 저장하고, 터미널 또는 콘솔에 저장된 위치로 이동합니다.

4. 다음 curl 명령을 실행하여 GeoFence를 생성합니다:

   ```sh
   curl --request POST 'https://atlas.microsoft.com/mapData/upload?api-version=1.0&dataFormat=geojson&subscription-key=<subscription_key>' \
        --header 'Content-Type: application/json' \
        --include \
        --data @geofence.json
   ```

   `<subscription_key>` 을 Azure Maps 계정의 API key를 가지고 있는 URL로 바꿉니다.

   URL은 `https://atlas.microsoft.com/mapData/upload` API를 통해 지도 데이터를 업로드하는데 사용됩니다. 호출에는 사용할 Azure Maps API를 지정하는 `api-version` 매개 변수가 포함되어 있습니다. 이는 API가 시간이 지남에 따라 변경될 수 있지만 이전 버전과의 호환성을 유지하기 위한 것입니다. 업로드되는 데이터 형식은 `geojson`로 설정됩니다.

   이것은 API 업로드를 위한 POST 요청을 실행하고 `location`이라고 부르는 응답 헤더 목록을 반환합니다.

   ```output
   content-type: application/json
   location: https://us.atlas.microsoft.com/mapData/operations/1560ced6-3a80-46f2-84b2-5b1531820eab?api-version=1.0
   x-ms-azuremaps-region: West US 2
   x-content-type-options: nosniff
   strict-transport-security: max-age=31536000; includeSubDomains
   x-cache: CONFIG_NOCACHE
   date: Sat, 22 May 2021 21:34:57 GMT
   content-length: 0
   ```

   > 🎓 웹 엔드포인트를 호출할 때, `key=value`와 같은 키 값 쌍 뒤에 `?`를 더해서 매개 변수를 전달할 수 있습니다. 키 값 쌍은 `&`로 구분됩니다.

5. Azure Maps 은 이를 즉시 처리하지 않으므로, `location` 헤더에 제공된 URL을 사용하여 업로드 요청이 완료되었는지 확인해야 합니다. 상태를 보려면 이 위치에 GET 요청을 만드십시오. `location` URL 끝에 subscription key `&subscription-key=<subscription_key>` 와 같이 추가하고, `<subscription_key>` 를 Azure Maps 계정의 API 키로 변경합니다. 다음 명령을 실행합니다:

   ```sh
   curl --request GET '<location>&subscription-key=<subscription_key>'
   ```

   `<location>` 를 `location` 헤더 값으로 변경합니다. 그리고 `<subscription_key>` 를 Azure Maps 계정의 API 키 값으로 변경합니다.

6. 응답에서 `status` 값을 확인하십시오. `Succeeded`가 아닌 경우, 잠시 기다렸다가 다시 시도하십시오.

7. 상태가 `Succeeded`로 돌아오면, 응답에서 `resourceLocation` 를 확인하십시오. 여기에는 GeoJson 객체의 고유한 ID (UDID라고 함)에 대한 세부 정보가 포함되어 있습니다. UDID 는 `metadata/` 뒤에 오는 값dmfh `api-version`을 포함하지 않습니다. 예를 들어, `resourceLocation` 이 다음과 같은 경우:

   ```json
   {
     "resourceLocation": "https://us.atlas.microsoft.com/mapData/metadata/7c3776eb-da87-4c52-ae83-caadf980323a?api-version=1.0"
   }
   ```

   그러면 UDID는 `7c3776eb-da87-4c52-ae83-caadf980323a` 입니다.

   이 UDID 의 사본을 보관하십시오. 지오펜스를 테스트하는데 필요한 것입니다.

## Test points against a geofence

Once the polygon has been uploaded to Azure Maps, you can test a point to see if it is inside or outside the geofence. You do this by making a web API request, passing in the UDID of the geofence, and the latitude and longitude of the point to test.

When you make this request, you can also pass a value called the `searchBuffer`. This tells the Maps API how accurate to be when returning results. The reason for this is GPS is not perfectly accurate, and sometimes locations can be out by meters if not more. The default for the search buffer is 50m, but you can set values from 0m to 500m.

When results are returned from the API call, one of the parts of the result is a `distance` measured to the closest point on the edge of the geofence, with a positive value if the point is outside the geofence, negative if it is inside the geofence. If this distance is less than the search buffer, the actual distance is returned in meters, otherwise the value is 999 or -999. 999 means that the point is outside the geofence by more than the search buffer, -999 means it is inside the geofence by more than the search buffer.

![A geofence with a 50m search buffer around in](../../../images/search-buffer-and-distance.png)

In the image above, the geofence has a 50m search buffer.

- A point in the center of the geofence, well inside the search buffer has a distance of **-999**
- A point well outside the search buffer has a distance of **999**
- A point inside the geofence and inside the search buffer, 6m from the geofence, has a distance of **6m**
- A point outside the geofence and inside the search buffer, 39m from the geofence, has a distance of **39m**

It is important to know the distance to the edge of the geofence, and combine this with other information such as other GPS readings, speed and road data when making decisions based off a vehicle location.

For example, imagine GPS readings showing a vehicle was driving along a road that ends up running next to a geofence. If a single GPS value is inaccurate and places the vehicle inside the geofence, despite there being no vehicular access, then it can be ignored.

![A GPS trail showing a vehicle passing the Microsoft campus on the 520, with GPS readings along the road except for one on the campus, inside a geofence](../../../images/geofence-crossing-inaccurate-gps.png)

In the above image, there is a geofence over part of the Microsoft campus. The red line shows a truck driving along the 520, with circles to show the GPS readings. Most of these are accurate and along the 520, with one inaccurate reading inside the geofence. There is no way that reading can be correct - there are no roads for the truck to suddenly divert from the 520 onto campus, then back onto the 520. The code that checks this geofence will need to take the previous readings into consideration before acting on the results of the geofence test.

✅ What additional data would you need to check to see if a GPS reading could be considered correct?

### Task - test points against a geofence

1. Start by building the URL for the web API query. The format is:

   ```output
   https://atlas.microsoft.com/spatial/geofence/json?api-version=1.0&deviceId=gps-sensor&subscription-key=<subscription-key>&udid=<UDID>&lat=<lat>&lon=<lon>
   ```

   Replace `<subscription_key>` with the API key for your Azure Maps account.

   Replace `<UDID>` with the UDID of the geofence from the previous task.

   Replace `<lat>` and `<lon>` with the latitude and longitude that you want to test.

   This URL uses the `https://atlas.microsoft.com/spatial/geofence/json` API to query a geofence defined using GeoJSON. It targets the `1.0` api version. The `deviceId` parameter is required and should be the name of the device the latitude and longitude comes from.

   The default search buffer is 50m, and you can change this by passing an additional parameter of `searchBuffer=<distance>`, setting `<distance>` to the search buffer distance in meters, form 0 to 500.

1. Use curl to make a GET request to this URL:

   ```sh
   curl --request GET '<URL>'
   ```

   > 💁 If you get a response code of `BadRequest`, with an error of:
   >
   > ```output
   > Invalid GeoJSON: All feature properties should contain a geometryId, which is used for identifying the geofence.
   > ```
   >
   > then your GeoJSON is missing the `properties` section with the `geometryId`. You will need to fix up your GeoJSON, then repeat the steps above to re-upload and get a new UDID.

1. The response will contain a list of `geometries`, one for each polygon defined in the GeoJSON used to create the geofence. Each geometry has 3 fields of interest, `distance`, `nearestLat` and `nearestLon`.

   ```output
   {
       "geometries": [
           {
               "deviceId": "gps-sensor",
               "udId": "7c3776eb-da87-4c52-ae83-caadf980323a",
               "geometryId": "1",
               "distance": 999.0,
               "nearestLat": 47.645875,
               "nearestLon": -122.142713
           }
       ],
       "expiredGeofenceGeometryId": [],
       "invalidPeriodGeofenceGeometryId": []
   }
   ```

   - `nearestLat` and `nearestLon` are the latitude and longitude of a point on the edge of the geofence that is closest to the location being tested.

   - `distance` is the distance from the location being tested to the closest point on the edge of the geofence. Negative numbers mean inside the geofence, positive outside. This value will be less than 50 (the default search buffer), or 999.

1. Repeat this multiple times with locations inside and outside the geofence.

## Use geofences from serverless code

You can now add a new trigger to your Functions app to test the IoT Hub GPS event data against the geofence.

### Consumer groups

As you will remember from previous lessons, the IoT Hub will allow you to replay events that have been received by the hub but not processed. But what would happen if multiple triggers connected? How will it know which one has processed which events.

The answer is it can't! Instead you can define multiple separate connections to read off events, and each one can manage the replay of unread messages. These are called _consumer groups_. When you connect to the endpoint, you can specify which consumer group you want to connect to. Each component of your application will connect to a different consumer group

![One IoT Hub with 3 consumer groups distributing the same messages to 3 different functions apps](../../../images/consumer-groups.png)

In theory up to 5 applications can connect to each consumer group, and they will all receive messages when they arrive. It's best practice to have only one application access each consumer group to avoid duplicate message processing, and ensure when restarting all queued messages are processed correctly. For example, if you launched your Functions app locally as well as running it in the cloud, they would both process messages, leading to duplicate blobs stored in the storage account.

If you review the `function.json` file for the IoT Hub trigger you created in an earlier lesson, you will see the consumer group in the event hub trigger binding section:

```json
"consumerGroup": "$Default"
```

When you create an IoT Hub, you get the `$Default` consumer group created by default. If you want to add an additional trigger, you can add this using a new consumer group.

> 💁 In this lesson, you will use a different function to test the geofence to the one used to store the GPS data. This is to show how to use consumer groups and separate the code to make it easier to read and understand. In a production application there are many ways you might architect this - putting both on one function, using a trigger on the storage account to run a function to check the geofence, or using multiple functions. There is no 'right way', it depends on the rest of your application and your needs.

### Task - create a new consumer group

1. Run the following command to create a new consumer group called `geofence` for your IoT Hub:

   ```sh
   az iot hub consumer-group create --name geofence \
                                    --hub-name <hub_name>
   ```

   Replace `<hub_name>` with the name you used for your IoT Hub.

1. If you want to see all the consumer groups for an IoT Hub, run the following command:

   ```sh
   az iot hub consumer-group list --output table \
                                  --hub-name <hub_name>
   ```

   Replace `<hub_name>` with the name you used for your IoT Hub. This will list all the consumer groups.

   ```output
   Name      ResourceGroup
   --------  ---------------
   $Default  gps-sensor
   geofence  gps-sensor
   ```

> 💁 When you ran the IoT Hub event monitor in an earlier lesson, it connected to the `$Default` consumer group. This was why you can't run the event monitor and an event trigger. If you want to run both, then you can use other consumer groups for all your function apps, and keep `$Default` for the event monitor.

### Task - create a new IoT Hub trigger

1. Add a new IoT Hub event trigger to your `gps-trigger` function app that you created in an earlier lesson. Call this function `geofence-trigger`.

   > ⚠️ You can refer to [the instructions for creating an IoT Hub event trigger from project 2, lesson 5 if needed](../../../2-farm/lessons/5-migrate-application-to-the-cloud/README.md#create-an-iot-hub-event-trigger).

1. Configure the IoT Hub connection string in the `function.json` file. The `local.settings.json` is shared between all triggers in the Function App.

1. Update the value of the `consumerGroup` in the `function.json` file to reference the new `geofence` consumer group:

   ```json
   "consumerGroup": "geofence"
   ```

1. You will need to use the subscription key for your Azure Maps account in this trigger, so add a new entry to the `local.settings.json` file called `MAPS_KEY`.

1. Run the Functions App to ensure it is connecting and processing messages. The `iot-hub-trigger` from the earlier lesson will also run and upload blobs to storage.

   > To avoid duplicate GPS readings in blob storage, you can stop the Functions App you have running in the cloud. To do this, use the following command:
   >
   > ```sh
   > az functionapp stop --resource-group gps-sensor \
   >                     --name <functions_app_name>
   > ```
   >
   > Replace `<functions_app_name>` with the name you used for your Functions App.
   >
   > You can restart it later with the following command:
   >
   > ```sh
   > az functionapp start --resource-group gps-sensor \
   >                     --name <functions_app_name>
   > ```
   >
   > Replace `<functions_app_name>` with the name you used for your Functions App.

### Task - test the geofence from the trigger

Earlier in this lesson you used curl to query a geofence to see if a point was located inside or outside. You can make a similar web request from inside your trigger.

1. To query the geofence, you need its UDID. Add a new entry to the `local.settings.json` file called `GEOFENCE_UDID` with this value.

1. Open the `__init__.py` file from the new `geofence-trigger` trigger.

1. Add the following import to the top of the file:

   ```python
   import json
   import os
   import requests
   ```

   The `requests` package allows you to make web API calls. Azure Maps doesn't have a Python SDK, you need to make web API calls to use it from Python code.

1. Add the following 2 lines to the start of the `main` method to get the Maps subscription key:

   ```python
   maps_key = os.environ['MAPS_KEY']
   geofence_udid = os.environ['GEOFENCE_UDID']
   ```

1. Inside the `for event in events` loop, add the following to get the latitude and longitude from each event:

   ```python
   event_body = json.loads(event.get_body().decode('utf-8'))
   lat = event_body['gps']['lat']
   lon = event_body['gps']['lon']
   ```

   This code converts the JSON from the event body to a dictionary, then extracts the `lat` and `lon` from the `gps` field.

1. When using `requests`, rather than building up a long URL as you did with curl, you can use just the URL part and pass the parameters as a dictionary. Add the following code to define the URL to call and configure the parameters:

   ```python
   url = 'https://atlas.microsoft.com/spatial/geofence/json'

   params = {
       'api-version': 1.0,
       'deviceId': 'gps-sensor',
       'subscription-key': maps_key,
       'udid' : geofence_udid,
       'lat' : lat,
       'lon' : lon
   }
   ```

   The items in the `params` dictionary will match the key value pairs you used when calling the web API via curl.

1. Add the following lines of code to call the web API:

   ```python
   response = requests.get(url, params=params)
   response_body = json.loads(response.text)
   ```

   This calls the URL with the parameters, and gets back a response object.

1. Add the following code below this:

   ```python
   distance = response_body['geometries'][0]['distance']

   if distance == 999:
       logging.info('Point is outside geofence')
   elif distance > 0:
       logging.info(f'Point is just outside geofence by a distance of {distance}m')
   elif distance == -999:
       logging.info(f'Point is inside geofence')
   else:
       logging.info(f'Point is just inside geofence by a distance of {distance}m')
   ```

   This code assumes 1 geometry, and extracts the distance from that single geometry. It then logs different messages based off the distance.

1. Run this code. You will see in the logging output if the GPS coordinates are inside or outside the geofence, with a distance if the point is within 50m. Try this code with different geofences based off the location of your GPS sensor, try moving the sensor (for example tethered to WiFi from a mobile phone, or with different coordinates on the virtual IoT device) to see this change.

1. When you are ready, deploy this code to your Functions app in the cloud. Don't forget to deploy the new Application Settings.

   > ⚠️ You can refer to [the instructions for uploading Application Settings from project 2, lesson 5 if needed](../../../2-farm/lessons/5-migrate-application-to-the-cloud/README.md#task---upload-your-application-settings).

   > ⚠️ You can refer to [the instructions for deploying your Functions app from project 2, lesson 5 if needed](../../../2-farm/lessons/5-migrate-application-to-the-cloud/README.md#task---deploy-your-functions-app-to-the-cloud).

> 💁 You can find this code in the [code/functions](code/functions) folder.

---

## 🚀 Challenge

In this lesson you added one geofence using a GeoJSON file with a single polygon. You can upload multiple polygons at the same time, as long as they have different `geometryId` values in the `properties` section.

Try uploading a GeoJSON file with multiple polygons and adjust your code to find which polygon the GPS coordinates are closest to or in.

## Post-lecture quiz

[Post-lecture quiz](https://black-meadow-040d15503.1.azurestaticapps.net/quiz/28)

## Review & Self Study

- Read more on geofences and some of their use cases on the [Geofencing page on Wikipedia](https://en.wikipedia.org/wiki/Geo-fence).
- Read more on Azure Maps geofencing API on the [Microsoft Azure Maps Spatial - Get Geofence documentation](https://docs.microsoft.com/rest/api/maps/spatial/getgeofence?WT.mc_id=academic-17441-jabenn).
- Read more on consumer groups in the [Features and terminology in Azure Event Hubs - Event consumers documentation on Microsoft docs](https://docs.microsoft.com/azure/event-hubs/event-hubs-features?WT.mc_id=academic-17441-jabenn#event-consumers)

## Assignment

[Send notifications using Twilio](assignment.md)
