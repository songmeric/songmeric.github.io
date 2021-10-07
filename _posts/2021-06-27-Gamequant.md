---
layout: post
title:  "Timeline video analysis of LoL replay - 1"
date:   2021-06-27 16:40:00
categories: gamequant

---

1. **리플레이 영상으로부터 시간 추출**
- **2.1** 전체 영상에서 현재 게임 시간 영역 추출 (빨간색 박스)

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled.png](/images/0627_1.png)

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%201.png](/images/0627_2.png)

- **2.2** 분과 초를 나타내는 각각의 영역으로 구분

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%202.png](/images/0627_3.png)

    ```python
    def extract_time_region(frame):
      x1_min, x1_max = 678, 684
      x2_min, x2_max = 684, 690
      x3_min, x3_max = 693, 699
      x4_min, x4_max = 699, 705

      y_min, y_max = 52, 61

      return [frame[y_min:y_max, x1_min:x1_max].copy(), frame[y_min:y_max, x2_min:x2_max].copy(), frame[y_min:y_max, x3_min:x3_max].copy(), frame[y_min:y_max, x4_min:x4_max].copy()]
    ```

- **2.3** 각 숫자를 나타내는 이미지 확보

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%203.png](/images/0627_4.png)

- **2.4 2.2**에서 얻은 각 영역을 **2.3**에서 확보한 숫자 이미지와 비교하여 현재 시간을 숫자(int type)로 변환. 비교에는 SSIM 사용

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%204.png](/images/0627_5.png)

    ```python
    def get_current_time(frame, digit_dict, ssim_threshold=0.95):
      time_digits = extract_time_region(frames[45])

      current_time = []
      for digit in time_digits:
        for name, digit_base in digit_dict.items():
          ssim_score = structural_similarity(np.pad(digit, ((3,3), (3,3), (0,0))), np.pad(digit_base, ((3,3), (3,3), (0,0))), multichannel=True)
          if ssim_score > ssim_threshold:
            current_time.append(name)
            break

      return current_time
    ```

- **2.5** 타임라인에 해당하는 영역 추출

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%205.png](/images/0627_6.png)

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%206.png](/images/0627_7.png)

    ```python
    def extract_timeline_region(frame):
      x_min, x_max = 517, 920
      y_min, y_max = 675, 715

      return frame[y_min:y_max, x_min:x_max].copy()
    ```

- **2.6 2.5**에서 추출한 타임라인 이미지에서 현재 프레임의 위치를 나타내는 노란색 바(bar)의 좌표 추출

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%207.png](/images/0627_8.png)

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%208.png](/images/0627_9.png)

    ```python
    def extract_timebar_region(timeline_region):
      timeline_region = timeline_region[10:20].copy()

      bar_region = np.all((timeline_region) > [75,  80, 85] ,2)
      exclue_region = np.invert(np.all(timeline_region > [100,  50, 50], 2))

      return bar_region & exclue_region
    ```

- **2.7 2.6**에서 얻은 현재 프레임을 나타내는 좌표와 **2.4**에서 얻은 현재 프레임의 게임 시간을 바탕으로 전체 게임 시간(seconds) 계산. 이를 활용하여 타임라인 중 특정 영역의 시간 또한 계산 가능. 

- 아래 예시에 사용된 게임의 경우, 실제 게임 시간은 25분 26초 (약 1526초)인데 구현된 방법을 통해 계산한 게임 시간은 25분 21초 (1521초)로 약 5초의 차이를 보임.

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%209.png](/images/0627_10.png)

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%2010.png](/images/0627_11.png)

    ```python
    def calc_the_full_time(current_time, timebar_region):
      '''
      if current time is 04:25, the input parameter 'current_time' is a list which contains [0, 4, 2, 5].
      '''
      current_time_secs = current_time[0]*600 + current_time[1]*60 + current_time[2]*10 + current_time[3]

      full_x_idx = timebar_region.shape[1]
      timebar_x_idx = np.mean(np.where(timebar_region)[1])
      full_time_secs = int(current_time_secs * (full_x_idx / timebar_x_idx))

      return full_time_secs
    ```
    
2. **타임라인 킬 로그** 
- 레드팀 킬 로그 바(kill log bar) 추출

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%2011.png](/images/0627_12.png)

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%2012.png](/images/0627_13.png)

- 블루팀 킬 로그 바(kill log bar) 추출

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%2013.png](/images/0627_14.png)

    ![Daily%20Report%202021%2006%2027%20(Sun)%208dcc1fcf5a4648d4baea556f3eb75f73/Untitled%2014.png](/images/0627_15.png)

- Code

```python
def extract_red_kill_bars(timeline_region):
  red_bars = timeline_region[2:20].copy()
  kill_idcs = (np.sum(red_bars[:5], 0 )[:,1] > 150) & (np.sum(red_bars[:5], 0 )[:,1] < 180)

  return red_bars, kill_idcs

def extract_blue_kill_bars(timeline_region):
  blue_bars = timeline_region[20:38].copy()
  kill_idcs = (np.sum(blue_bars[:5], 0 )[:,1] > 200) & (np.sum(blue_bars[:5], 0 )[:,1] < 300)

  return blue_bars, kill_idcs
```

- 한타라고 판단하는 기준(e.g., 몇분 간격 사이에 킬이 n개 발생하면 한타로 가정)만 설정하면 바로 적용 가능.
