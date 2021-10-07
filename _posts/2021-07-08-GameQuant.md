---
layout: post
title:  "2021-07-08: Line Push statistics enhancements and First Gank predictions"
date:   2021-07-08 16:40:00
categories: gamequant

---

1. **Line Push Statistics**
- 1) 라인 위치 좌표 지도에 바텀 라인 추가 (블루팀 기준) 
- 노란색: 미는 라인
- 진녹색: 중간에 걸친 라인
- 초록색: 당기는 라인

    ![Daily%20Report%202021%2007%2008%20(Thu)%2003d84b7fb812460d9e332f734a7cce42/line.png](/images/line.png)

    ```python
    def get_line_push_map(size):
      line_map = np.zeros(size)
      middle_push_line_map = np.zeros(size)

      # Top line 
      cv2.line(line_map, (28,50), (44, 35), 2, 13)
      cv2.line(line_map, (48, 31), (60, 20), 3, 13)
      cv2.line(middle_push_line_map, (32, 25), (60, 50), 1, 5)

      # Mid Line
      cv2.line(line_map, (95,120), (116,101), 2, 13)
      cv2.line(line_map, (118,100), (131,89), 3, 13)
      cv2.line(middle_push_line_map, (98,87), (128,117), 1, 5)

      line_map[(middle_push_line_map == 1) & (line_map > 0)] = 1

      # Bot Line
      cv2.line(line_map, (160, 190), (185, 175), 2, 13)
      cv2.line(line_map, (190, 170), (200, 150), 3, 13)
      cv2.line(middle_push_line_map, (177,163), (195,180), 1, 5)

      line_map[(middle_push_line_map == 1) & (line_map > 0)] = 1
      return line_map
    ```

- 2) csv 좌표 정보를 line push 정보로 변환
- push_df: 각 프레임 당 해당 라인의 챔피언이 라인 상에 어디에 존재하는지를 나타낸 값
- push_probs: push_df를 바탕으로 각 챔피언이 라인을 어떻게 관리하는지에 대한 확률값 (푸쉬할 확률, 당길 확률, 중간에 걸칠 확률 순서)

    ![Daily%20Report%202021%2007%2008%20(Thu)%2003d84b7fb812460d9e332f734a7cce42/_2021-07-08__7.59.00.png](/images/_2021-07-08__7.59.00.png)

- 사용된 코드

```python
def get_push_label(x, y, line_push_map):
    if np.isnan(x):
        return -1

    else:
        x, y = int(x), int(y)
        return line_push_map[y, x]

def get_bot_push_label(x1, y1, x2, y2, line_push_map):
    if np.isnan(x1):
        return -1

    else:
        x, y = int((x1+x2)/2), int((y1+y2)/2)
        return line_push_map[y, x]

def calc_push_probs(push_df):
  '''
  return: push_prob, pull_prob, center_prob
  '''
  push_df_target = push_df[push_df > 0].copy()
  return (push_df_target.value_counts() / len(push_df_target)).values

def convert_xy_to_push_df(csv_df, line_push_map):
  blue_top_push_df = csv_df_smooth[['1p_x_coord', '1p_y_coord']].apply(lambda xy : get_push_label(*xy, line_push_map), axis=1)
  blue_mid_push_df = csv_df_smooth[['3p_x_coord', '3p_y_coord']].apply(lambda xy : get_push_label(*xy, line_push_map), axis=1)
  blue_bot_push_df = csv_df_smooth[['4p_x_coord', '4p_y_coord', '5p_x_coord', '5p_y_coord']].apply(lambda xy : get_bot_push_label(*xy, line_push_map), axis=1)

  push_df = pd.DataFrame([blue_top_push_df, blue_mid_push_df, blue_bot_push_df]).T
  push_df.columns = ['blue_top_push', 'blue_mid_push', 'blue_bot_push']

  top_push_probs = calc_push_probs(blue_top_push_df) 
  mid_push_probs = calc_push_probs(blue_mid_push_df) 
  bot_push_probs = calc_push_probs(blue_bot_push_df)

  return push_df, top_push_probs, mid_push_probs, bot_push_probs
```

2. **First Ganking Prediction**
    - 고려 중인 변수 리스트: 
    1) 탑, 미드, 바텀 라인 푸쉬 확률 (밀, 당길, 중간에 걸칠 확률, 총 9개) 
    2) 10개의 챔피언 ID (총 10개, one-hot encoding으로 하면 차원이 너무 커져서 해당 문제 해결 방법 고려 필요)
    3) 상대 정글러가 첫 동선에서 먹는 평균 정글 캠프 개수 (1개)
    - 상대 정글의 첫번째 동선 (탑,미드,바텀 갱 혹은 카운터 정글)을 분류(classification)로 풀 수 있을 것 같으며 random forest나 boosting 계열의 모델 사용 가능
