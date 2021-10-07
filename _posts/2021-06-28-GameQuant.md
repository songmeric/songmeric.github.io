---
layout: post
title:  "Timeline video analysis of LoL replay(cont'd) and Objective tracking from minimap"
date:   2021-06-28 16:40:00
categories: gamequant

---

1. **타임라인 오브젝트 로그**
- **1.1** 오브젝트 아이콘 이미지 확보 ([링크](https://drive.google.com/drive/folders/10rVLw9o3UFm54wRwKctGNuWhMmyilBLc?usp=sharing))

    ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled.png](/images/0628_0.png)

- **1.2** 타임라인 상의 오브젝트 아이콘과 **1.1**에서 확보한 아이콘 매칭 (아래는 레드팀만 시각화)

    ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%201.png](/images/0628_1.png)

    ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%202.png](/images/0628_2.png)

    ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%203.png](/images/0628_3.png)

    ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%204.png](/images/0628_4.png)

    ```python
    object_log = dict()
    timeline_W = timeline.shape[1]

    for obj_name, obj_icon in object_dict.items():
        (tH, tW) = obj_icon.shape[:2]

        # Find the matched region using the query icon as a template
        result = cv2.matchTemplate(timeline, obj_icon, cv2.TM_CCORR_NORMED)
        scores = np.sort(result.reshape(-1))[::-1]

        # for top-k matched regions
        for score in scores[:10]:
            (yCoords, xCoords) = np.where(result == score)

            for x, y in zip(xCoords, yCoords):
                matched_part = timeline[y:y+tH, x:x+tW].copy()
                ssim = structural_similarity(obj_icon, matched_part, multichannel=True)

                if ssim > 0.7:
                    x_center = x+tW//2

                    object_time_secs = int(x_center / timeline_W * full_time_secs)

                    if obj_name in object_log:
                        object_log[obj_name].append(object_time_secs)                
                    else:
                        object_log[obj_name] = [object_time_secs]
    ```

- 구현된 방법으로 구한 오브젝트별 타임라인

    ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%205.png](/images/0628_5.png)

- 실제 경기 기록에서 얻은 오브젝트별 타임라인
- blue_dragon: 11.43 | 17.09 |
- blue_inhib: 32.35
- blue_tower: 24.19 | 26.06
- red_baron: 21.05 | 30.26 |

2. **Minion tracking**
    1. Minion tracking을 위한 각 진영별 미니언 아이콘 확보
    - 생각보다 미니언이 실제 좌표 상에서 나타나는 모양이 다양한 것 같음. 예를 들어, 여러 미니언이 겹치게 되면 미니맵 상에 여러 개의 원이 겹치게 되는데 이러한 경우를 모두 고려하여 robust하게 잘 잡아내는 방법이 필요해보임. 모든 미니언 아이콘 하나하나를 정확하게 잡아내는 것은 현실적으로 어려워보이며 교착 상태의 미니언을 잡아내는 것이 중요하므로 해당 상황이 미니맵 상에서 가지는 패턴을 찾아내는 등의 아이디어가 필요.
      
        ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%206.png](/images/0628_6.png)
        
    2. 예시 1: 미드의 미니언은 잡아내지 못하는 모습
       
        ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%207.png](/images/0628_7.png)
        
    3. **예시 2**: 정작 중요한 탑 라인에서 라인이 걸친 경우를 잡아내지 못하는 모습
       
        ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%208.png](/images/0628_8.png)
    
3. **정글몹 tracking**
    1. tracking을 위한 정글몹 아이콘 확보
       
        ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%209.png](/images/0628_9.png)
        
    2. 매칭 결과 시각화
       
        ![Daily%20Report%202021%2006%2028%20(Mon)%20043d27826fbd4f5190df78462996a6db/Untitled%2010.png](/images/0628_10.png)
