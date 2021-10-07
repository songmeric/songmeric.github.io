---
layout: post
title:  "2021-07-07 Line Push statistics"
date:   2021-07-07 16:40:00
categories: gamequant

---

1. **Line Push Statistics 결과**
- 탑 미드를 위주로 해당 챔피언이 라인을 중간에서 유지하는지(1), 당기는지(2) 혹은 미는지(3)에 대한 통계값 추출 시스템 완성
- 1) 탑 미드 라인의 챔피언이 라인을 밀고 있는지 여부를 확인하기 위한  좌표 지도 (블루팀 기준) 
- 노란색: 미는 라인
- 진녹색: 중간에 걸친 라인
- 초록색: 당기는 라인

    ![Daily%20Report%202021%2007%2007%20(Wed)%204a2f3fcc69764261ac1a9d5ac1fd1606/Untitled.png](/images/0707_0.png)

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

      return line_map
    ```

- 2) Champion tracking을 통해 얻은 챔피언의 프레임별 좌표값을 통해 1)에서 구한 지도상 어디에 위치하는지에 대한 값으로 변환
- 값: -1 (미니맵 상에 존재하지 않는 경우)
- 값: 0 (라인 상에 존재하지 않는 경우)
- 값: 1 (중간에 걸친 라인)
- 값: 2 (당기는 라인)
- 값: 3 (미는 라인)

- 아래의 예시는 탑 라인의 챔피언의 좌표값들을 line push label로 변환한 결과

    ![Daily%20Report%202021%2007%2007%20(Wed)%204a2f3fcc69764261ac1a9d5ac1fd1606/Untitled%201.png](/images/0707_1.png)

- 3) 2)에서 구한 값들을 바탕으로 라인 상에 존재할 때 라인을 어떻게 관리하는지에 대한 확률값으로 변환 (값이 1, 2, 3인 경우만 고려) 

- 아래 예시의 경우에는 해당 챔피언이 라인을 미는(3) 확률이 46.5%, 당길 확률(2)이 31.9% 그리고 중간에 걸치게 유지할 확률을 21.5%로 해석할 수 있다.

    ![Daily%20Report%202021%2007%2007%20(Wed)%204a2f3fcc69764261ac1a9d5ac1fd1606/Untitled%202.png](/images/0707_2.png)
