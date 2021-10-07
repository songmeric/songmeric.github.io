---
layout: post
title:  "Jg mobs/minions tracking enhancement from minimap"
date:   2021-06-29 16:40:00
categories: gamequant

---

1. **정글몹 tracking**
    1. 이전에 정글몹 아이콘을 이용한 tracking 방법 대체. 정글몹은 미니맵 상에서 고정된 위치에 존재하므로 해당 위치에 정글몹 아이콘이 존재하는지 여부를 바탕으로 tracking
    2. 각 정글몹별 미니맵 상의 좌표 (수작업으로 구한 값이라 오차가 있을 수 있음.)
       
        ```python
        jg_pos_dict = {
            # Name : [x_min, y_min, x_max, y_max]
            'blue_team_gromp' : [27, 93, 31, 97],
            'blue_team_blue' : [50, 100, 56, 106],
            'blue_team_wolf' : [51, 122, 55, 126],
            'blue_team_wraith' : [100, 140, 103, 143],
            'blue_team_red' : [111, 161, 115, 165],
            'blue_team_golem' : [119, 178, 123, 182],
        
            'red_team_gromp' : [182, 123, 186, 127],
            'red_team_blue' : [158, 116, 164, 122],
            'red_team_wolf' : [160, 95, 163, 98],
            'red_team_wraith' : [110, 77, 113, 80],
            'red_team_red' : [99, 57, 103, 61],
            'red_team_golem' : [91, 37, 95, 41],
        
            'upper_scuttler' : [61, 77, 64, 80],
            'lower_scuttler' : [150, 142, 154, 146],
        
            'dragon' : [142, 151, 148, 158],
            'baron' : [67, 63, 74, 70],
        }
        ```
        
    3. **2**에서 구한 좌표에 정글몹이 존재하는 여부를 확인하는 코드
    - 정글몹 아이콘에 해당하는 HSV 범위(jg_gen_min_HSV, jg_gen_max_HSV)를 미리 정하고 해당 범위 안에 HSV 값이 들어오는지 여부를 바탕으로 리젠 여부 확인   
      
        ```python
        jg_gen_min_HSV = [10, 90, 90]
        jg_gen_max_HSV = [26, 220, 200]
        
        # For each frame
        jg_gen_rows = []
        jg_names = jg_pos_dict.keys()
        
        for frame in frames:
          jg_gen_row = []
        
          # Get minimap part from the entire frame
          minimap = extract_minimap(frame)
          minimap_clone = minimap.copy()
        
          # For each icon which exist in the current game
          for name in jg_names:
            x_min, y_min, x_max, y_max = jg_pos_dict[name]
            jg_region = minimap[y_min:y_max, x_min:x_max].copy()
            matched_HSV = cv2.cvtColor(cv2.resize(jg_region, (1,1)), cv2.COLOR_BGR2HSV)[0,0].astype(np.float32)
        
            is_gen = int(np.all(jg_gen_min_HSV < matched_HSV) & np.all(matched_HSV < jg_gen_max_HSV))
            jg_gen_row.append(is_gen)
        
          jg_gen_rows.append(jg_gen_row)
        
        jg_gen_df = pd.DataFrame(jg_gen_rows, columns=jg_names)
        ```
        
    4. 단순히 HSV 값을 바탕으로 정글몹 리젠 여부를 따지면 몇몇 프레임에서 오류가 발생하는 경우 확인. 이를 해결하기 위하여 앞뒤 일정 간격의 프레임을 비교하여 오류라고 판단되는 패턴을 보일 경우, 앞뒤 프레임의 값으로 대체하여 오류 해결. 
       
        ```python
        def replace_weird_values(jg_gen_df):
          patterns = [[1,0,1], [1,0,0,1], [1,0,0,0,1], [1,0,0,0,0,1], [0,1,0], [0,1,1,0], [0,1,1,1,0], [0,1,1,1,1,0]]
        
          for i in range(jg_gen_df.shape[1]):
            for pattern in patterns:
              idcs = np.where([np.all(v.values == pattern) for v in jg_gen_df.iloc[:,i].rolling(len(pattern))])[0]
        
              for idx in idcs:
                jg_gen_df.iloc[idx-len(pattern)+1:idx+1,i] = pattern[0]
        ```
        
    5. 위와 같은 방법으로 구현한 정글몹 tracking 결과 시각화 (빨간색: 젠 O / 파란색: 젠 X)
       
        ![Daily%20Report%202021%2006%2029%20(Tue)%205c8d12253eb74f109ec58e632183ab2a/ezgif.com-gif-maker.gif](/images/0629_gif_0.gif)
    
2. **미니언 tracking**
   
    1. 미니언 경로 추출
    - 미니언은 일정한 경로 상에서 움직이므로 이를 나타내는 지도 확보.
    - 경로 상에 존재하는 타워 아이콘이 미니언으로 착각될 수 있으므로 해당 부분 제거
    - 좌표를 수작업으로 찾은 것이므로 오차가 발생할 가능성 존재 
      
        ![Daily%20Report%202021%2006%2029%20(Tue)%205c8d12253eb74f109ec58e632183ab2a/minion.png](/images/0629_minion.png)
        
        ```python
        def get_line_map(minimap):
          line_map = np.zeros((map_y_max-map_y_min, map_x_max-map_x_min))
        
          # Top line
          line_map[51:140, 13:18] = 1
          line_map[18:23, 45:150] = 1
          cv2.line(line_map, (14,50), (45, 23), 1, 4)
        
          # Top Towers
          line_map[60:76, 13:17] = 0
          line_map[118:127, 15:18] = 0
          line_map[18:23, 55:65] = 0
          line_map[18:23, 110:120] = 0
        
          # Mid Line
          cv2.line(line_map, (57,158), (158,62), 1, 4)
        
          # Mid Towers
          line_map[142:150, 67:80] = 0
          line_map[87:100, 124:133] = 0
          line_map[70:80, 140:145] = 0
        
          # Bot Line
          line_map[200:205, 67:160] = 1
          line_map[74:170, 196:201] = 1
          cv2.line(line_map, (160,202), (198, 170), 1, 4)
        
          # Bot Towers
          line_map[200:204, 96:103] = 0
          line_map[200:205, 146:159] = 0
          line_map[145:156, 197:201] = 0
          line_map[93:105, 196:198] = 0
        
          return line_map
        ```
        
    2. 경로 상에 존재하는 미니언 영역 추출
    - LAB 색상값을 바탕으로 1에서 구한 경로 상에 미니언이 존재하는 여부 확인 
      
        ![Daily%20Report%202021%2006%2029%20(Tue)%205c8d12253eb74f109ec58e632183ab2a/minion_2.png](/images/0629_minion_2.png)
        
        ```python
        def get_red_minion_map(minimap, line_map):
          # Red minion
          red_minion_min_LAB = [60, 170, 150]
          red_minion_max_LAB = [125, 190, 165]
        
          red_minion_map = np.all(red_minion_min_LAB < cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB), 2) & np.all(cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB) < red_minion_max_LAB, 2)
          red_minion_map = red_minion_map & line_map.astype(bool)
        
          return red_minion_map
          
        def get_blue_minion_map(minimap, line_map):
          # Blue minion
          blue_minion_min_LAB = [80, 110, 90]
          blue_minion_max_LAB = [140, 120, 110]
        
          blue_minion_map = np.all(blue_minion_min_LAB < cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB), 2) & np.all(cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB) < blue_minion_max_LAB, 2)
          blue_minion_map = blue_minion_map & line_map.astype(bool)
        
          return blue_minion_map
        ```
        
    3. 2에서 구한 각 진영 미니언의 위치를 바탕으로 교착 상태의 중심점 추출
        - 2에서 구한 미니언 맵에 dilation morphology를 적용하여 미니언 웨이브 덩어리로 묶기
          
            ![Daily%20Report%202021%2006%2029%20(Tue)%205c8d12253eb74f109ec58e632183ab2a/Untitled.png](/images/0629_0.png)
            
        - scikit-image를 활용하여 각 웨이브 덩어리의 centroid 좌표값 추출 후, centroid 사이의 거리를 계산하여 '교착 상태에 빠진 양쪽 미니언 웨이브 사이의 거리 (dist_threshold)'보다 가까운 centroid 탐색.
          
            ```python
            from skimage.measure import label, regionprops
            
            def get_minion_lock(red_minion_map, blue_minion_map, dist_threshold=25):
              # Dilate
              kernel = np.ones((7,7), np.uint8)
              red_minion_map_di = cv2.dilate(red_minion_map.astype(np.uint8).copy(), kernel, iterations=1)
              blue_minion_map_di = cv2.dilate(blue_minion_map.astype(np.uint8).copy(), kernel, iterations=1)
            
              # Centroids
              red_labels = label(red_minion_map_di)
              red_centroids = [np.array(prop.centroid) for prop in regionprops(red_labels)]
              
              blue_labels = label(blue_minion_map_di)
              blue_centroids = [np.array(prop.centroid) for prop in regionprops(blue_labels)]
            
              lock_pts = []
              for r_cen in red_centroids:
                for b_cen in blue_centroids:
                  dist = np.linalg.norm(r_cen - b_cen)
            
                  if dist < dist_threshold:
                    lock_pts.append([int((r_cen[1]+b_cen[1])/2), int((r_cen[0]+b_cen[0])/2)])
            
              return lock_pts
            ```
        
    4. 3에서 구한 중심점 시각화 결과
       
        ![Daily%20Report%202021%2006%2029%20(Tue)%205c8d12253eb74f109ec58e632183ab2a/ezgif.com-gif-maker_(1).gif](/images/0629_gif_1.gif)
