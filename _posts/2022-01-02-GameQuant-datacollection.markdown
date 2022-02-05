---
layout: post
title:  "Large scale LoL match data collection without production API Key"
date:   2022-01-02 16:40:00
categories: gamequant

---

Pick and Ban phase는 사람이 진행해야 공평하다. 체스 랭크 게임을 AI가 플레이 하는 것은 핵이나 다름없지 않겠는가?

GameQuant의 개발을 위해서 라이엇에서 정식으로 API key를 발급받고 테라바이트 단위의 콜을 하기는 쉽지 않을 것이다. 
우리에게 주어진 것은 Call rate limit이 뚜렷하게 걸려있는 personal key들 뿐.

허락을 받기 보단 용서를 빌으라 했던가, 일단은 데이터를 모아보았다.

1. **라이엇 계정 생성 매크로** 를 사용하여 500개가 넘는 계정을 자동으로 생성하였다. 이메일 인증이 되지 않은 계정은 API키를 발급받을 수 없는 관계로 fake email generator를 사용하여
생성된 이메일이 사라지기 전 가입 후 인증 절차까지 전부 자동화 하였다.
  

2. API 키는 24시간에 한번씩 직접 클릭해 리셋시켜줘야만 그 access가 유지된다. 이 또한 selenium을 통해 해결했다. 물론 라이엇 측에서도 몇가지 방어체계를 구축 해 두었지만, anti captcha 앞에서는 
힘없이 무너져 버렸다. 코드는 일부만 공개하겠다.
   
    ```python
    from selenium import webdriver
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support.ui import WebDriverWait
    from selenium.webdriver.chrome.service import Service
    from anticaptchaofficial.recaptchav2proxyless import *
    import threading

    t0 = time.time()
    allocation = 15
    with open('/home/ubuntu/gamequant/24.csv','r') as f:
        data = f.readlines()


    def collect(start):
        global allocation
        options = webdriver.ChromeOptions()
        options.add_argument("--headless")
        options.add_argument("--no-sandbox")
        options.add_argument("--disable-gpu")

        s = Service('/home/ubuntu/gamequant/chromedriver')
        browser = webdriver.Chrome(service=s, options=options)

        browser.get('domain')
        end = start + allocation
        if end >= len(data):
            end = len(data)
        for k, i in enumerate(data[start:end]):
            try:
                i = i[:-1]
                # filling form
                id = i.split(',')[0]
                password = i.split(',')[1]
                WebDriverWait(browser, 120).until(lambda x: x.find_element(By.XPATH,
                                                                           '//*[@id="site-navbar-collapse"]/ul[2]/li/a'))
                buttons = browser.find_elements(By.XPATH, '//*[@id="site-navbar-collapse"]/ul[2]/li/a')
                for button in buttons:
                    button.click()

                WebDriverWait(browser,120).until(lambda x: x.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/div[2]/div/div/div/div[1]/div/input'))
                browser.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/div[2]/div/div/div/div[1]/div/input').send_keys(id)
                browser.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/div[2]/div/div/div/div[2]/div/input').send_keys(password)

                # a = ActionChains(browser)
                browser.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/button').click()
                # WebDriverWait(browser, 10).until(EC.frame_to_be_available_and_switch_to_it((By.CSS_SELECTOR, "iframe[name^='a-'][src^='https://www.google.com/recaptcha/api2/anchor?']")))
                # WebDriverWait(browser, 10).until(EC.element_to_be_clickable((By.XPATH, "//span[@id='recaptcha-anchor']"))).click()

               //중간 부분 코드 삭제
                browser.execute_script(f'document.getElementById("g-recaptcha-response").innerHTML="{g_response}";')
                browser.find_element(By.XPATH, '/html/body/div[2]/div/form/div[3]/div/div[3]/div[2]/div[2]/input').click()

                new_api_key = browser.find_element(By.ID, 'apikey').get_attribute('value')
                browser.get('logout')


                print(new_api_key)
                # print(LolWatcher(new_api_key).summoner.by_name("kr", 'username')["puuid"])
                with open('/home/ubuntu/gamequant/api.csv', 'a') as f:
                    f.writelines(new_api_key+','+i+'\n')
            except Exception as E:
                print('##########Error with', start, k, i, E)

    Pros = []
    if __name__ == "__main__":
        # change the value inside the range
        n = 4
        for i in range(n):
            print("Thread Started")
            p = threading.Thread(target=collect, args=(i*allocation,))
            Pros.append(p)
            p.start()

        for t in Pros:
            t.join()
        print(time.time()-t0)
    ```
    
3. 수백개의 계정을 동원해 24시간에 한번씩 리셋시키며 AWS EC2 최강의 인스턴스들을 돌려가며 멀티쓰레딩으로 데이터를 모았다. 코드는 악용의 여지가 있으니 역시 아래 일부만 공개.

    ```python
    from riotwatcher import LolWatcher
    import threading
    import time
    from selenium import webdriver
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support.ui import WebDriverWait
    from selenium.webdriver.chrome.service import Service
    from anticaptchaofficial.recaptchav2proxyless import *

    ###### DEALING WITH ERROR ######
    # If there are too many 'error_puuid' printed, try one of these:
    # (1) change match to match_v5 in when calling api
    # (2) change match_v5 to match in when calling api
    # (3) check whether the apis are valid uncommenting the section 'error_puuid(3)'


    # to measure the time taken (comparing effectiveness of threading)
    t0 = time.time()

    # Initial settings

    # number of api key used
    num_api_key = 15
    num_summoner = 9573
    # reading files
    with open('/home/ubuntu/gamequant/api.csv', 'r') as f:
        data = f.readlines()
    print(data)

    apis = []
    idpw = []

    for i in data:
        i = i.split(',')
        idpw.append(i[1:])
        apis.append(i[0])

    with open("/home/ubuntu/gamequant/summoner_processed.csv") as f:
        names = f.readlines()



    #API Collector
    def api_collect(id, password):
        global allocation
        options = webdriver.ChromeOptions()
        options.add_argument("--headless")
        options.add_argument("--no-sandbox")
        options.add_argument("--disable-gpu")

        s = Service('/home/ubuntu/gamequant/chromedriver')
        browser = webdriver.Chrome(service=s, options=options)

        browser.get('https://developer.riotgames.com')


        try:
            WebDriverWait(browser, 120).until(lambda x: x.find_element(By.XPATH,
                                                                       '//*[@id="site-navbar-collapse"]/ul[2]/li/a'))
            buttons = browser.find_elements(By.XPATH, '//*[@id="site-navbar-collapse"]/ul[2]/li/a')
            for button in buttons:
                button.click()

            WebDriverWait(browser,120).until(lambda x: x.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/div[2]/div/div/div/div[1]/div/input'))
            browser.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/div[2]/div/div/div/div[1]/div/input').send_keys(id)
            browser.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/div[2]/div/div/div/div[2]/div/input').send_keys(password)

            # a = ActionChains(browser)
            browser.find_element(By.XPATH, '/html/body/div/div/div/div[2]/div/div/button').click()
            # WebDriverWait(browser, 10).until(EC.frame_to_be_available_and_switch_to_it((By.CSS_SELECTOR, "iframe[name^='a-'][src^='https://www.google.com/recaptcha/api2/anchor?']")))
            # WebDriverWait(browser, 10).until(EC.element_to_be_clickable((By.XPATH, "//span[@id='recaptcha-anchor']"))).click()

            # token = browser.find_element(By.XPATH, '/html/body/input').get_attribute('value')
            # print(token)
            # wait for "solved" selector to come up
            g_response = 0
            //코드 중간부분 삭제
            browser.execute_script(f'document.getElementById("g-recaptcha-response").innerHTML="{g_response}";')
            browser.find_element(By.XPATH, '/html/body/div[2]/div/form/div[3]/div/div[3]/div[2]/div[2]/input').click()

            new_api_key = browser.find_element(By.ID, 'apikey').get_attribute('value')
            browser.get('https://developer.riotgames.com/logout')


            print(new_api_key)
            return new_api_key
        except Exception as E:
            print('##########Error with', E)


    def collect(api_start, name_start, name_end):
        global apis, names
        # Making Dictionary to put api_keys
        config={}
        for i in range(num_api_key):
            config[str(i)] =''


        # Setting initial values (api setting, summoner name setting)
        api_end = api_start + num_api_key
        for i, api in enumerate(apis[api_start:api_end]):
            try:
                Api = LolWatcher(api)
                answer = Api.summoner.by_name(~~~~)
                config[list(config.keys())[i]] = api
            except:
                print('API_ERROR',i,api)
                print(idpw[i+api_start])
                Id, pwd = idpw[i+api_start][0], idpw[i+api_start][1]
                temp_api = api_collect(Id, pwd[:-1])
                print(temp_api)
                config[list(config.keys())[i]] = temp_api

        api = LolWatcher(config['0'])
        num = 0
        api_call = 0
        thread_num = int(api_start/num_api_key)
        if name_end > len(names):
            name_end = len(names)

        # MAIN LOOP
        for k, i in enumerate(names[name_start:name_end]):
            x = 0
            # 23h stop
            '''
            if time.time() - t1 > 43200:
                while text != 'A':
                    t1 = time.time()
                    print(api_start / num_api_key, k, i)
                    text = input(f'{thread_num}th Thread 23h: Press "A" to continue\n')
                text = 'B'
                print(f'{thread_num}th 23h Thread started')
                with open('api.csv') as f:
                    apis = f.readlines()
                print(apis)
                for i, api in enumerate(apis[api_start:api_end]):
                    try:
                        Api = LolWatcher(api)
                        answer = @@*!&#(*!*
                        config[list(config.keys())[i]] = api
                    except:
                        print('API_ERROR', i, api)
            '''
            summoner_name = i[:-1]
            while True:
                try:
                    if (api_call < 99):
                        answer = api.summoner.by_name("kr", '123123')["puuid"]
                        api_call += 1
                        break
                    elif (api_call >= 99):
                        num += 1
                        x = num % len(list(config.keys()))
                        api = LolWatcher(config[list(config.keys())[x]])
                        api_call = 0
                        answer = api.summoner.by_name("kr", '125152131')["puuid"]
                        print(f"{thread_num}, api_swap")
                        break
                except:
                    Id, pwd = idpw[api_start + x][0], idpw[api_start + x][1]
                    temp_api = api_collect(Id, pwd[:-1])
                    config[list(config.keys())[x]] = temp_api
                    num += 1
                    x = num % len(list(config.keys()))
                    api = LolWatcher(config[list(config.keys())[x]])
                    api_call = 0

            try:
                if(api_call < 99):
                    summoner_id = api.summoner.by_name("kr", summoner_name)["puuid"]
                    print(summoner_id)
                    api_call += 1
                elif(api_call >= 99):
                    num += 1
                    x = num % len(list(config.keys()))
                    api = LolWatcher(config[list(config.keys())[x]])
                    api_call = 0
                    summoner_id = api.summoner.by_name("kr", summoner_name)["puuid"]
                    api_call += 1
                    print(f"{thread_num}, api_swap")

                start = 0
                matchlist = []
                while True:
                    if (api_call < 99):
                      /통째로 삭제, 한개의 아이디 당 API Call 횟수를 제한시켜 에러를 방지하였음.
                            break
                print(len(matchlist))

            except:
                print(f"error_puuid{thread_num}", i)
                continue


            # Extracting matchinfo from the games in matchlist
            for gameId in matchlist:
                while(True):
                    try:
                        if(api_call < 99):
                            matchinfo = api.match_v5.by_id("ASIA", gameId)
                            api_call += 1
                        elif(api_call >= 99):
                            num += 1
                            x = num % len(list(config.keys()))
                            api = LolWatcher(config[list(config.keys())[x]])
                            api_call = 0
                            matchinfo = api.match_v5.by_id("ASIA", gameId)
                            api_call += 1
                            print(f"{thread_num}, api_swap")


                        if matchinfo["info"]["queueId"] != 420:
                            # print(matchinfo["info"]["queueId"])
                            # print(num)
                            continue

                        participants = matchinfo["info"]["participants"]
                        # print(participants[0]["teamId"])
                        # print(participants[0]["win"])

                        if (participants[0]["teamId"] == 100 and participants[0]["win"] == True) or (
                                participants[0]["teamId"] != 100 and participants[0]["win"] != True):
                            Win = 1
                        else:
                            Win = 0

                        Blue = []
                        Red = []
                        Ban = [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
                        K = matchinfo["info"]["teams"][0]["bans"]
                        # print(K)
                        for i in K:
                            Ban[i["pickTurn"] - 1] = i["championId"]

                        K = matchinfo["info"]["teams"][1]["bans"]
                        for i in K:
                            Ban[i["pickTurn"] - 1] = i["championId"]

                        for i in range(10):
                            if participants[i]["teamId"] == 100:
                                Blue.append(participants[i]["championId"])
                            elif participants[i]["teamId"] == 200:
                                Red.append(participants[i]["championId"])
                            else:
                                print("Error team Id does not exist: " + "teamId =" + participants[i]["teamId"])

                        if Blue[0] == Blue[1]:
                            continue

                        with open("/home/ubuntu/gamequant/match_data_ban.csv", "a") as f:
                            line = str([gameId]) + ","  + str(Ban) + "," + str(Blue) + "," + str(Red) + "," + str([Win]) + "\n"
                            f.write(line)

                        with open("/home/ubuntu/gamequant/match_data_ban_total.csv", 'a') as f:
                            line = str(matchinfo) + '\n'
                            f.write(line)

                        # print(gameId)
                        # print(api_call)
                        break
                    except Exception as e:
                        print(f"error pass{thread_num}",num, e)
                        while True:
                            try:
                                if (api_call < 99):
                                    answer = api.summoner.by_name("kr", '124124')["puuid"]
                                    api_call += 1
                                    break
                                elif (api_call >= 99):
                                    num += 1
                                    x = num % len(list(config.keys()))
                                    api = LolWatcher(config[list(config.keys())[x]])
                                    api_call = 0
                                    answer = api.summoner.by_name("kr", '231312313')["puuid"]

                                    print(f"{thread_num}, api_swap")
                                    break
                            except:
                                Id, pwd = idpw[api_start + x][0], idpw[api_start + x][1]
                                temp_api = api_collect(Id, pwd[:-1])
                                config[list(config.keys())[x]] = temp_api
                                num += 1
                                x = num % len(list(config.keys()))
                                api = LolWatcher(config[list(config.keys())[x]])
                                api_call = 0

            print('Success', thread_num, k, i)
        print(f'Finished {thread_num}')


    Pros = []
    if __name__ == "__main__":
        # change the value inside the range
        n = len(apis) // num_api_key
        for i in range(n):
            print("Thread Started")
            p = threading.Thread(target=collect, args=(i*num_api_key, i*num_summoner, (i+1)*num_summoner,))
            Pros.append(p)
            p.start()

        for t in Pros:
            t.join()
        print(time.time()-t0)
    ```
    
    
    꽤 힘들었지만 AWS EC2, RunCommand등을 사용하여 수십개의 인스턴스들을 동시에 병렬로 다루며 large data pipeline 구축에 대해서 많은 것을 배웠다. 
    어떻게 이 데이터를 처리했으며 어떤식으로 최종 상품을 만드는데 사용하였는지 등은 블로깅 하고싶지 않으므로 비밀에 부쳐두려 한다.
