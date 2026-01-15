# Google Travel에서 호텔 스クレイピング하기

[![Promo](https://github.com/bright-kr/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/) 

이 가이드는 Selenium 방식 또는 Bright Data의 API를 사용하여 Google Travel에서 호텔 목록, 가격 및 편의시설을 수집하는 방법을 설명합니다.

- [Prerequisites](#prerequisites)
- [What To Extract From Google Travel](#what-to-extract-from-google-travel)
- [Extracting The Data With Selenium](#extracting-the-data-with-selenium)
- [Extracting the Data With Bright Data’s Travel API](#extracting-the-data-with-bright-datas-travel-api)
    - [Requests](#requests)
    - [AIOHTTP](#aiohttp)
- [Bright Data’s Alternative Solutions](#bright-datas-alternative-solutions)

## Prerequisites

여행 데이터를 스크레이핑하려면 Python과 Selenium, Requests 또는 AIOHTTP 모듈 중 하나가 필요합니다. Selenium을 사용하면 Google Travel에서 호텔 정보를 직접 스크레이핑합니다. Requests와 AIOHTTP를 사용하면 Bright Data의 [Booking.com API](https://brightdata.co.kr/products/web-scraper/booking)를 사용하게 됩니다.

Selenium을 사용하는 경우 [webdriver](https://googlechromelabs.github.io/chrome-for-testing/)가 설치되어 있는지 확인합니다. Selenium이 익숙하지 않다면, 빠르게 익히기 위해 [이 가이드](https://brightdata.co.kr/blog/how-tos/using-selenium-for-web-scraping)를 확인해 보시기 바랍니다.

Selenium 설치:

```
pip install selenium
```

Requests 설치:

```
pip install requests
```

AIOHTTP 설치:

```bash
pip install aiohttp
```

## What To Extract From Google Travel

모든 호텔 결과는 Google Travel의 커스텀 `c-wiz` 요소에 포함되어 있습니다.

![Inspect c-wiz Element](https://brightdata.co.kr/wp-content/uploads/2025/01/image-32.png)

하지만 페이지에는 많은 `c-wiz` 요소가 존재합니다. 각 호텔 카드에는 `div` 및 해당 `c-wiz` 요소에서 직접 하위로 내려오는 `a` 요소가 포함되어 있습니다. 이 요소들에서 하위에 있는 모든 `a` 태그를 찾기 위해 CSS selector를 작성할 수 있습니다: `c-wiz > div > a`.

![Inspect a Element](https://brightdata.co.kr/wp-content/uploads/2025/01/image-33.png)

리스팅의 이름은 `h2`에 포함되어 있습니다.

![Inspect h2 Element](https://brightdata.co.kr/wp-content/uploads/2025/01/image-34.png)

가격은 `span`에 포함되어 있습니다.

![Inspect Price Element](https://brightdata.co.kr/wp-content/uploads/2025/01/image-35.png)

편의시설은 `li`(리스트) 요소에 포함되어 있습니다.

![Inspect Amenities](https://brightdata.co.kr/wp-content/uploads/2025/01/image-36.png)

호텔 카드를 찾은 후에는, 위에서 언급한 모든 데이터를 해당 카드에서 추출할 수 있습니다.

## Extracting The Data With Selenium

Selenium으로 이 데이터를 추출하는 작업은 무엇을 찾아야 하는지 알고 나면 비교적 간단합니다. 하지만 Google Travel은 결과를 동적으로 로드하므로, 사전 구성된 대기(wait), 마우스 클릭, 커스텀 창에 의해 겨우 유지되는 섬세한 프로세스가 됩니다.

아래는 전체 Python 스크립트입니다:

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.action_chains import ActionChains
import json
from time import sleep

OPTIONS = webdriver.ChromeOptions()
OPTIONS.add_argument("--headless")
OPTIONS.add_argument("--window-size=1920,1080")



def scrape_hotels(location, pages=5):
    driver = webdriver.Chrome(options=OPTIONS)
    actions = ActionChains(driver)
    url = f"https://www.google.com/travel/search?q={location}"
    driver.get(url)
    done = False

    found_hotels = []
    page = 1
    result_number = 1
    while page <= pages:
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        sleep(5)
        hotel_links = driver.find_elements(By.CSS_SELECTOR, "c-wiz > div > a")
        print(f"-----------------PAGE {page}------------------")
        print("FOUND ITEMS: ", len(hotel_links))
        for hotel_link in hotel_links:
            hotel_card = hotel_link.find_element(By.XPATH, "..")
            try:
                info = {}
                info["url"] = hotel_link.get_attribute("href")
                info["rating"] = 0.0
                info["price"] = "n/a"
                info["name"] = hotel_card.find_element(By.CSS_SELECTOR, "h2").text
                price_holder = hotel_card.find_elements(By.CSS_SELECTOR, "span")
                info["amenities"] = []
                amenities_holders = hotel_card.find_elements(By.CSS_SELECTOR, "li")
                for amenity in amenities_holders:
                    info["amenities"].append(amenity.text)
                if "DEAL" in price_holder[0].text or "PRICE" in price_holder[0].text:
                    if price_holder[1].text[0] == "$":
                        info["price"] = price_holder[1].text
                else:
                    info["price"] = price_holder[0].text
                rating_holder = hotel_card.find_elements(By.CSS_SELECTOR, "span[role='img']")
                if rating_holder:
                    info["rating"] = float(rating_holder[0].get_attribute("aria-label").split(" ")[0])
                info["result_number"] = result_number
                
                if info not in found_hotels:
                    found_hotels.append(info)
                result_number+=1
                
            except:
                continue
        print("Scraped Total:", len(found_hotels))
        
        next_button = driver.find_elements(By.XPATH, "//span[text()='Next']")
        if next_button:
            print("next button found!")
            sleep(1)
            actions.move_to_element(next_button[0]).click().perform()
            page+=1
            sleep(5)
        else:
            done = True

    driver.quit()

    with open("scraped-hotels.json", "w") as file:
        json.dump(found_hotels, file, indent=4)

if __name__ == "__main__":
    PAGES = 2
    scrape_hotels("miami", pages=PAGES)
```

스크립트가 수행하는 작업을 단계별로 살펴보겠습니다:

1. 먼저 `ChromeOptions`의 인스턴스를 생성합니다. 이를 사용하여 `--headless` 및 `--window-size=1920,1080` 인자를 추가합니다.

> **Note**\
> 커스텀 window size가 없으면 결과가 제대로 로드되지 않으며, 동일한 결과를 반복해서 스크레이핑하게 됩니다.

2. 브라우저를 실행할 때 키워드 인자 `options=OPTIONS`를 사용합니다. 이를 통해 커스텀 옵션이 적용된 Chrome이 실행됩니다.

3. `ActionChains(driver)`는 `ActionChains` 인스턴스를 제공합니다. 이는 이후 스크립트에서 커서를 `Next` 버튼으로 이동한 다음 클릭하는 데 사용합니다.

4. 런타임을 포함하기 위해 `while` 루프를 사용합니다. 스크레이핑이 완료되면 이 루프를 종료합니다.

5. `hotel_links = driver.find_elements(By.CSS_SELECTOR, "c-wiz > div > a")`는 페이지의 모든 호텔 링크를 제공합니다. xpath를 사용해 부모 요소를 찾습니다: `hotel_card = hotel_link.find_element(By.XPATH, "..")`.

6. 앞서 확인한 개별 데이터 조각을 모두 추출합니다:
    - url: `hotel_link.get_attribute("href")`
    - name: `hotel_card.find_element(By.CSS_SELECTOR, "h2").text`
    - 가격을 찾을 때 카드에 `DEAL`, `GREAT PRICE`와 같은 추가 요소가 포함되는 경우가 있습니다. 항상 올바른 가격을 가져오기 위해 `span` 요소를 배열로 추출합니다. 배열에 이러한 단어가 포함되면 첫 번째 요소(`price_holder[0].text`)가 아니라 두 번째 요소(`price_holder[1].text`)를 사용합니다.
    - 평점을 찾을 때도 `find_elements()` 메서드를 사용합니다. 평점이 없으면 기본값으로 `n/a`를 부여합니다.
    - `hotel_card.find_elements(By.CSS_SELECTOR, "li")`는 편의시설 요소들을 제공합니다. 각 요소의 `text` 속성을 사용해 추출합니다.
7. 원하는 모든 페이지를 스크레이핑할 때까지 이 루프를 계속합니다. 데이터를 확보하면 `done`을 `True`로 설정하고 루프를 종료합니다.
8. 브라우저를 닫고 `json.dump()`를 사용하여 스크레이핑한 모든 데이터를 JSON 파일로 저장합니다.

## Extracting the Data With Bright Data’s Travel API

스크레이퍼에 의존하거나 selector 및 locator를 다루고 싶지 않다면, [travel data](https://brightdata.co.kr/use-cases/travel)를 사용하거나 [Booking.com API](https://brightdata.co.kr/products/web-scraper/booking)를 사용하여 호텔 데이터를 추출할 수 있습니다. 이를 수행하는 두 가지 방법은 `requests` 모듈과 AIOHTTP 라이브러리입니다.

### Requests

아래 코드는 Booking.com API를 사용할 수 있도록 설정합니다. API key, 여행 location, check-in 날짜 및 check-out 날짜를 입력하기만 하면 됩니다. 먼저 API에 요청을 보내 데이터를 생성합니다. 그런 다음 리포트가 준비될 때까지 10초마다 데이터를 반복적으로 확인합니다. 데이터를 수신하면 JSON 파일로 저장합니다.

```python
import requests
import json
import time


def get_bookings(api_key, location, dates):
    url = "https://api.brightdata.com/datasets/v3/trigger"

    #booking.com dataset
    dataset_id = "gd_m4bf7a917zfezv9d5"

    endpoint = f"{url}?dataset_id={dataset_id}&include_errors=true"
    auth_token = api_key

    #
    headers = {
        "Authorization": f"Bearer {auth_token}",
        "Content-Type": "application/json"
    }

    payload = [
        {
            "url": "https://www.booking.com",
            "location": location,
            "check_in": dates["check_in"],
            "check_out": dates["check_out"],
            "adults": 2,
            "rooms": 1
        }
    ]

    response = requests.post(endpoint, headers=headers, json=payload)

    if response.status_code == 200:
        print("Request successful. Response:")
        print(json.dumps(response.json(), indent=4))
        return response.json()["snapshot_id"]
    else:
        print(f"Error: {response.status_code}")
        print(response.text)

def poll_and_retrieve_snapshot(api_key, snapshot_id, output_file="snapshot-data.json"):
    #create the snapshot url
    snapshot_url = f"https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}?format=json"
    headers = {
        "Authorization": f"Bearer {api_key}"
    }

    print(f"Polling snapshot for ID: {snapshot_id}...")

    while True:
        response = requests.get(snapshot_url, headers=headers)
        
        if response.status_code == 200:
            print("Snapshot is ready. Downloading...")
            snapshot_data = response.json()
            #write the snapshot to a new json file
            with open(output_file, "w", encoding="utf-8") as file:
                json.dump(snapshot_data, file, indent=4)
            print(f"Snapshot saved to {output_file}")
            break
        elif response.status_code == 202:
            print("Snapshot is not ready yet. Retrying in 10 seconds...")
        else:
            print(f"Error: {response.status_code}")
            print(response.text)
            break
        
        time.sleep(10)


if __name__ == "__main__":
    
    API_KEY = "your-bright-data-api-key"
    LOCATION = "Miami"
    CHECK_IN = "2025-02-01T00:00:00.000Z"
    CHECK_OUT = "2025-02-02T00:00:00.000Z"
    DATES = {
        "check_in": CHECK_IN,
        "check_out": CHECK_OUT
    }
    snapshot_id = get_bookings(API_KEY, LOCATION, DATES)
    poll_and_retrieve_snapshot(API_KEY, snapshot_id)
```

- `get_bookings()`는 `API_KEY`, `LOCATION`, `DATES`를 받습니다. 그런 다음 데이터 요청을 수행하고 `snapshot_id`를 반환합니다.
- `snapshot_id`는 snapshot을 조회하는 데 필요합니다.
- `snapshot_id`가 생성된 후, `poll_and_retrieve_snapshot()`는 데이터가 준비되었는지 10초마다 확인합니다.
- 데이터가 준비되면 `json.dump()`를 사용해 JSON 파일로 저장합니다.

코드를 실행하면 터미널에서 아래와 유사한 내용을 확인할 수 있습니다.

```
Request successful. Response:
{
    "snapshot_id": "s_m5moyblm1wikx4ntot"
}
Polling snapshot for ID: s_m5moyblm1wikx4ntot...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is ready. Downloading...
Snapshot saved to snapshot-data.json
```

그런 다음 아래와 같은 객체로 가득 찬 JSON 파일을 얻게 됩니다.

```json
{
        "input": {
            "url": "https://www.booking.com",
            "location": "Miami",
            "check_in": "2025-02-01T00:00:00.000Z",
            "check_out": "2025-02-02T00:00:00.000Z",
            "adults": 2,
            "rooms": 1
        },
        "url": "https://www.booking.com/hotel/us/ramada-plaze-by-wyndham-marco-polo-beach-resort.html?checkin=2025-02-01&checkout=2025-02-02&group_adults=2&no_rooms=1&group_children=",
        "location": "Miami",
        "check_in": "2025-02-01T00:00:00.000Z",
        "check_out": "2025-02-02T00:00:00.000Z",
        "adults": 2,
        "children": null,
        "rooms": 1,
        "id": "55989",
        "title": "Ramada Plaza by Wyndham Marco Polo Beach Resort",
        "address": "19201 Collins Avenue",
        "city": "Sunny Isles Beach (Florida)",
        "review_score": 6.2,
        "review_count": "1788",
        "image": "https://cf.bstatic.com/xdata/images/hotel/square600/414501733.webp?k=4c14cb1ec5373f40ee83d901f2dc9611bb0df76490f3673f94dfaae8a39988d8&o=",
        "final_price": 217,
        "original_price": 217,
        "currency": "USD",
        "tax_description": null,
        "nb_livingrooms": 0,
        "nb_kitchens": 0,
        "nb_bedrooms": 0,
        "nb_all_beds": 2,
        "full_location": {
            "description": "This is the straight-line distance on the map. Actual travel distance may vary.",
            "main_distance": "11.4 miles from downtown",
            "display_location": "Miami Beach",
            "beach_distance": "Beachfront",
            "nearby_beach_names": []
        },
        "no_prepayment": false,
        "free_cancellation": true,
        "property_sustainability": {
            "is_sustainable": false,
            "level_id": "L0",
            "facilities": [
                "436",
                "490",
                "492",
                "496",
                "506"
            ]
        },
        "timestamp": "2025-01-07T16:43:24.954Z"
    },
```

### AIOHTTP

[AIOHTTP](https://brightdata.co.kr/blog/web-data/speed-up-web-scraping) 라이브러리를 사용하면 여러 데이터셋을 동시에 trigger, poll, download할 수 있으므로 이 프로세스가 더 빨라질 수 있습니다. 아래 코드는 위의 Requests 예제의 개념을 기반으로 하되, `aiohttp.ClientSession()`을 사용하여 여러 요청을 비동기적으로 수행합니다.

```python
import aiohttp
import asyncio
import json


async def get_bookings(api_key, location, dates):
    url = "https://api.brightdata.com/datasets/v3/trigger"
    dataset_id = "gd_m4bf7a917zfezv9d5"
    endpoint = f"{url}?dataset_id={dataset_id}&include_errors=true"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    payload = [
        {
            "url": "https://www.booking.com",
            "location": location,
            "check_in": dates["check_in"],
            "check_out": dates["check_out"],
            "adults": 2,
            "rooms": 1
        }
    ]

    async with aiohttp.ClientSession(headers=headers) as session:
        async with session.post(endpoint, json=payload) as response:
            if response.status == 200:
                response_data = await response.json()
                print(f"Request successful for location: {location}. Response:")
                print(json.dumps(response_data, indent=4))
                return response_data["snapshot_id"]
            else:
                print(f"Error for location: {location}. Status: {response.status}")
                print(await response.text())
                return None


async def poll_and_retrieve_snapshot(api_key, snapshot_id, output_file):
    snapshot_url = f"https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}?format=json"
    headers = {
        "Authorization": f"Bearer {api_key}"
    }

    print(f"Polling snapshot for ID: {snapshot_id}...")

    async with aiohttp.ClientSession(headers=headers) as session:
        while True:
            async with session.get(snapshot_url) as response:
                if response.status == 200:
                    print(f"Snapshot for {output_file} is ready. Downloading...")
                    snapshot_data = await response.json()
                    # Save snapshot data to a file
                    with open(output_file, "w", encoding="utf-8") as file:
                        json.dump(snapshot_data, file, indent=4)
                    print(f"Snapshot saved to {output_file}")
                    break
                elif response.status == 202:
                    print(f"Snapshot for {output_file} is not ready yet. Retrying in 10 seconds...")
                else:
                    print(f"Error polling snapshot for {output_file}. Status: {response.status}")
                    print(await response.text())
                    break

            await asyncio.sleep(10)


async def process_location(api_key, location, dates):
    snapshot_id = await get_bookings(api_key, location, dates)
    if snapshot_id:
        output_file = f"snapshot-{location.replace(' ', '_').lower()}.json"
        await poll_and_retrieve_snapshot(api_key, snapshot_id, output_file)


async def main():
    api_key = "your-bright-data-api-key"
    locations = ["Miami", "Key West"]
    dates = {
        "check_in": "2025-02-01T00:00:00.000Z",
        "check_out": "2025-02-02T00:00:00.000Z"
    }

    # Process all locations in parallel
    tasks = [process_location(api_key, location, dates) for location in locations]
    await asyncio.gather(*tasks)


if __name__ == "__main__":
    asyncio.run(main())
```

- 이제 `get_bookings()`와 `poll_and_retrieve_snapshot()` 모두 `aiohttp.ClientSession` 객체를 사용하여 서버에 대한 async 요청을 생성합니다.
- `process_location()`은 특정 location에 대한 모든 데이터를 처리하는 데 사용합니다.
- `main()`은 모든 location에 대해 `process_location()`을 동시에 호출할 수 있게 해줍니다.

아래는 출력 예시입니다:

```
Request successful for location: Miami. Response:
{
    "snapshot_id": "s_m5mtmtv62hwhlpyazw"
}
Request successful for location: Key West. Response:
{
    "snapshot_id": "s_m5mtmtv72gkkgxvdid"
}
Polling snapshot for ID: s_m5mtmtv62hwhlpyazw...
Polling snapshot for ID: s_m5mtmtv72gkkgxvdid...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is ready. Downloading...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot saved to snapshot-miami.json
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is ready. Downloading...
Snapshot saved to snapshot-key_west.json
```

## Bright Data’s Alternative Solutions

[Web Scraper APIs](https://brightdata.co.kr/products/web-scraper) 외에도, Bright Data는 다양한 요구를 충족하도록 맞춤 설계된 즉시 사용 가능한 데이터셋을 제공합니다. 가장 수요가 높은 여행 데이터셋은 다음과 같습니다:

- [Hotel Datasets](https://brightdata.co.kr/products/datasets/travel/hotels)
- [Expedia Datasets](https://brightdata.co.kr/products/datasets/travel/expedia)
- [Tourism Datasets](https://brightdata.co.kr/products/datasets/tourism)
- [Booking.com Datasets](https://brightdata.co.kr/products/datasets/booking)
- [TripAdvisor Datasets](https://brightdata.co.kr/products/datasets/tripadvisor)

완전 관리형 또는 자체 관리형 커스텀 데이터셋 중에서 선택할 수 있으며, 이를 통해 어떤 공개 웹사이트에서든 데이터를 추출하고 정확한 사양에 맞게 커스터마이징할 수 있습니다.