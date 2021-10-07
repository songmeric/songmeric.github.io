---
layout: post
title:  "Automated data collection pipeline"
date:   2021-10-01 16:40:00
categories: gamequant

---

1. LCU Client API
- LCU Driver, ffmpeg, replay API 등을 활용하여 제작한 auto data collector.
- 리플레이를 순차적으로 처음부터 끝까지 8배속으로 녹화하고, 미니맵 부분만 crop filter 적용,  저장공간을 절약한다.
- API documentation이 없어 꽤 애를 먹었지만, swagger로 콜 보내보면서 열심히 작업한 결과물이다.

    ```python
    import requests
    import json
    import time
    import os
    import ffmpeg
    from lcu_driver import Connector
    from requests.packages.urllib3.exceptions import InsecureRequestWarning
    requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

    HOST = 'https://127.0.0.1:2999'
    def get_gameInfo():
        r = requests.get(f'{HOST}/replay/playback', verify=False)
        data = r.text
        print(data)
        return json.loads(data)

    def start_recording(end_time):
        data = {
          "codec": "webm",
          "endTime": end_time,
          "enforceFrameRate": False,
          "framesPerSecond": 120,
          "lossless": True,
          "recording": True,
          "replaySpeed": 8,
          "startTime": 0,
          "path": "C:\\Users\\Administrator\\Desktop\\Replays"
        }
        r = requests.post(f'{HOST}/replay/recording', json=data, verify=False)
        data = r.text
        print(data)
        return json.loads(data)

    def get_record_status():
        r = requests.get(f'{HOST}/replay/recording', verify=False)
        data = r.text
        print('get_record_status :', data)
        return json.loads(data)

    connector = Connector()

    @connector.ready
    async def connect(connection):
        game_id_list = ['5480130224','5484336857','5484351788']

        for i in game_id_list:
            download = await connection.request('post', f'/lol-replays/v1/rofls/{i}/download', data={'gameId': i})
            print(download)
            time.sleep(8)
            play = await connection.request('post', f'/lol-replays/v1/rofls/{i}/watch', data={'gameId' : i})
            print(play)

            time.sleep(25)

            d = get_gameInfo()
            gameTime = d['length']
            print('[*] gameTime:', gameTime)

            d = start_recording(gameTime)
            recording = True
            while recording:
                d = get_record_status()
                recording = d['recording']
                percent = d['currentTime'] / d['endTime'] * 100
                print(f'percent : {percent:.2f}%')
                time.sleep(1)

            print("Complete")
            os.system('taskkill /f /im "League of Legends.exe"')
            original_file = f'C:\\Users\\Administrator\\Desktop\\Replays\\11-19_KR-{i}_01.webm'
            stream = ffmpeg.input(original_file)
            stream = ffmpeg.crop(stream, 1391, 692, 289, 289)
            stream = ffmpeg.output(stream, f'C:\\Users\\Administrator\\Desktop\\Replays\\{i}.webm')
            ffmpeg.run(stream)
            os.remove(original_file)


    @connector.close
    async def disconnect(connection):
        print('Finished task')


    # r = requests.get(f'{HOST}/replay/render', verify=False)
    # print(r.text)

    connector.start()
    ```
