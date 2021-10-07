---
layout: post
title:  "Daily Report 2021.06.22 (Tue)"
date:   2021-06-22 16:40:00
categories: gamequant

---

1. 샘플 영상 확보

2. 영상으로부터 **미니맵 영역 추출**
   
    ```python
    map_x_min = 1150
    map_x_max = 1370
    
    map_y_min = 490
    map_y_max = 715
    
    minimap = frame[map_y_min:map_y_max, map_x_min:map_x_max]
    ```
    
3. 영상으로부터 **해당 게임에 사용된 챔피언 종류 추출**
   
    ```python
    def get_all_icons_in_frame(frame):
        '''
        Icon size = 24 x 24
        return 10 icons which exist in the game
        
        '''
        
        ICON_SIZE = 24
        
        y_locs = [108, 175, 245, 312, 381]
        x_locs = [22, 1328]
        
        icons = []
        
        for x_loc in x_locs:
            for y_loc in y_locs:
                icon = frame[y_loc:y_loc+ICON_SIZE,
                                   x_loc:x_loc+ICON_SIZE].copy()
                
                icon_circle = np.zeros_like(icon)
                icon_circle = cv2.circle(icon_circle, (ICON_SIZE//2, ICON_SIZE//2), radius=ICON_SIZE//2+2, thickness=-1, color=(255,255,255))
                
                icon[np.invert(icon_circle.astype(bool))] = 255
                icons.append(icon)
        
        return icons
    ```
    
4. 전체 챔피언 **아이콘 이미지 확보**: [링크](https://drive.google.com/drive/folders/15lnIzRSaSQ5hU3aP9uIWP6hw67bh11Az?usp=sharing)
5. 3.에서 얻은 챔피언 아이콘과 4.에서 확보한 이미지 아이콘 사이의 **매칭을 통해 챔피언 이름 연결**
    - SSIM 기반의 similarity 측정
    
    ```python
    from skimage.metrics import structural_similarity
    
    icons = get_all_icons_in_frame(frames[0])
    for i, icon in enumerate(icons):
        # Initilization
        max_ssim = -10
        matched_name = None
        matched_icon = None
        
        for champ_name, base_icon in base_icons.items():
            ssim = structural_similarity(icon, base_icon, multichannel=True)
            
            if ssim > max_ssim:
                max_ssim = ssim
                matched_name = champ_name
                matched_icon = base_icon
        
        icon_dict[matched_name] = icon[3:-3, 3:-3]
    ```
    
6. 3.에서 얻은 각 챔피언의 **프레임 당 위치 예측 모델** (**JgPos v1.1**)
   
    ```python
    minimap = frame[map_y_min:map_y_max, map_x_min:map_x_max]
    minimap_clone = minimap.copy()
    
    for name, icon in icon_dict.items():
        icon_mini = cv2.resize(icon, (12,12))
        (tH, tW) = icon_mini.shape[:2]
    
        result = cv2.matchTemplate(minimap, icon_mini, cv2.TM_CCORR_NORMED)
        (yCoords, xCoords) = np.where(result == np.max(result))
    
        x, y = xCoords[0], yCoords[0]        
        matched_part = minimap_clone[y:y+tH, x:x+tW].copy()
        matched_ssim = structural_similarity(icon_mini, matched_part, multichannel=True)
    
        if matched_ssim > ssim_threshold:
            cv2.rectangle(minimap_clone, (x, y), (x + tW, y + tH), (0, 0, 255), 1)
            cv2.putText(minimap_clone, name, (x, y), cv2.FONT_HERSHEY_SIMPLEX, 0.3, (0,0,255))
    ```
    
    - **결과 시각화**
      
        ![Daily%20Report%202021%2006%2022%20(Tue)%20e67021740de945c19d805282a8335e23/JgPos_v1.1.gif](images/JgPos_v1.1.gif)
