---
layout: post
title:  "GameQuant Dev Diary: Image tracking and data collection pipeline - 4"
date:   2021-06-25 16:40:00
categories: gamequant

---

1. **JgPos v1.3** 개발
- 매칭의 대상이 되는 query icon **테두리에 각 진영(red, blue)에 해당하는 원을 추가**하여 매칭 정확도 향상

    + 추가로 미니맵 상의 아이콘은 row resolution임을 고려하여 **Gaussin Blur**처리 추가 (cv2.GaussianBlur(matched_icon, (3,3), 100))

    ![Daily%20Report%202021%2006%2025%20(Fri)%20b0ea09558b2a46758c89c7380838eadd/Untitled.png](/images/0625_5.png)

- **Hyper-parameter** 수정

    1) occlusion된 경우를 고려하여 영상의 일부를 crop하여 비교하는 search space 확장

    ![Daily%20Report%202021%2006%2025%20(Fri)%20b0ea09558b2a46758c89c7380838eadd/Untitled%201.png](/images/0625_4.png)

    2) 속도 향상을 위하여 matching score 기준 상위 3개(top-3)만 비교하도록 수정
    3) 성능 향상을 위하여 matching 기준 수정 ((HSV_diff < 25 and ssim > 0.5) or (ssim > 0.65))
    4) Center-cropping 제거

- **결과 시각화** (위: v1.3 / 아래: 성능 비교를 위한 v1.2)

    ![Daily%20Report%202021%2006%2025%20(Fri)%20b0ea09558b2a46758c89c7380838eadd/ezgif.com-gif-maker_(3).gif](/images/0625_2.gif)

    ![Daily%20Report%202021%2006%2025%20(Fri)%20b0ea09558b2a46758c89c7380838eadd/ezgif.com-gif-maker.gif](/images/0625_1.gif)
    
2. 성능 확인을 위한 **test case 추가**
- JgPos v1.3 시각화 결과

    - 문제 1. 귀환 모션 (키아나와 조이가 귀환하는 장면을 잡아내지 못함)
    - 해당 문제를 v1.4에서 개선할 계획

    ![Daily%20Report%202021%2006%2025%20(Fri)%20b0ea09558b2a46758c89c7380838eadd/ezgif.com-gif-maker_(4).gif](/images/0625_3.gif)
