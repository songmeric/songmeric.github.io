---
layout: post
title:  "Minion tracking noise interpolation."
date:   2021-07-01 16:40:00
categories: gamequant

---

1. **Minion Tracking**
- noise(중간중간 엉뚱한 위치에 예측) 제거와 중간 몇몇 프레임에서 누락되는 경우를 보완(interpolate)
- 기존 방식은 프레임마다 미니언의 위치를 예측하고 morphology dilation을 통해 미니언을 웨이브로 묶은 다음 블루팀 웨이브와 레드팀 웨이브가 가까이 있을 경우 교착 상태(lock)으로 좌표값 반환
- noise 제거와 일부 프레임 누락 문제 해결을 위하여 각 교착 상태의 좌표를 각각의 상황(위치 기준)으로 구분 
- 각 lock 좌표에 대해 기존 lock 좌표로 구성된 graph(하나의 상황을 묶은 것)와 비교하여 새로운 graph에 해당할 경우 새롭게 graph를 생성하고 그렇지 않으면 기존의 graph에 하나로 합치는 방법
- 이때 기존 그래프의 좌표와 비슷한 위치에 존재하지만 프레임 차이(frame_dist)가 큰 경우에는 같은 웨이브라고 보기 어려우므로 따로 구

    ```python
        max_dist = 10
        frame_dist = 30

            # merge minion locks based on the locations
        lock_graphs = []
        for frame, pts in lock_dict.items():
            for pt in pts:
                if len(lock_graphs) == 0:
                    lock_graph = {'base': pt, 'recent_frame': frame, frame: pt}
                    lock_graphs.append(lock_graph)
                else:
                    new_graph = True
                    for graph in lock_graphs:
                        base_pt = graph['base']
                        dist = np.sqrt((pt[0]-base_pt[0])**2 + (pt[1]-base_pt[1])**2)

                        if (dist < max_dist) and (frame - graph['recent_frame']) < frame_dist:
                            graph[frame] = pt
                            graph['recent_frame'] = frame
                            new_graph = False

                    if new_graph:
                        lock_graph = {'base': pt, 'recent_frame': frame, frame: pt}
                        lock_graphs.append(lock_graph)
    ```

- **노이즈 제거**
- 이전 과정에서 생성된 각 그래프를 구성하는 lock 좌표의 개수가 기준 (min_pts = 5) 보다 작을 경우에 해당 그래프는 노이즈로 간주하여 제거

    ```python
    # drop noise lock which has a small number of frames
        lock_graphs_wo_noise = []
        for graph in lock_graphs:
            if (len(graph)-2) > min_pts:
                lock_graphs_wo_noise.append(graph)
    ```

- **Interpolation**
- 챔피언 tracking과 같이 interpolate 적용 후 savgol_filter를 이용하여 smooth하게 만들어준다

    ```python
        # interpolate the intermediate frames
        lock_graph_dfs_itp = []
        for graph in lock_graphs_wo_noise:
            # dictionary to Dataframe
            pts_df = pd.DataFrame(graph).drop(['base', 'recent_frame'], 1).T

            # add NaN frame rows to Dataframe
            frame_idcs = pts_df.index
            pts_df_w_na = []

            for idx in range(min(frame_idcs), max(frame_idcs)+1):
                try:
                    pts_df_w_na.append(pts_df.loc[idx])
                except:
                    pts_df_w_na.append(pd.Series([np.nan, np.nan], name=idx))
            pts_df_w_na = pd.concat(pts_df_w_na, 1).T

            # Interpolate the NaN frame rows
            pts_df_itp = pts_df_w_na.interpolate()

            from scipy.signal import savgol_filter
            window_length = len(pts_df_itp) - min_pts

            if window_length % 2 == 0:
                window_length -= 1

            if window_length > 5:
                pts_df_itp[0] = savgol_filter(pts_df_itp[0], window_length=window_length, polyorder=3)
                pts_df_itp[1] = savgol_filter(pts_df_itp[1], window_length=window_length, polyorder=3)

            lock_graph_dfs_itp.append(pts_df_itp)
    ```

    ![Daily%20Report%202021%2007%2001%20(Thu)%20fa2b172c2d5c4093836d48a4b7fbcfbf/ezgif.com-gif-maker_(6).gif](/images/0701_6.gif)
    
2. **Minion Tracking 개선**
    - 하지만 단순히 미니언의 위치로만 교착 지점을 찾을 경우, 챔피언 아이콘에 의해 웨이브 전체 혹은 많은 부분이 가려지는 상황에는 tracking이 불가능하다는 문제점 발생
    - 여러가지 방법을 시도해보았으나 큰 효과가 없었고 아이디어를 고민해보다가 미니언을 가려버린다는 것은 결국 해당 챔피언이 미니언 표시를 대신하고 있다고 볼 수 있으므로, 각 라인 상(미니언 경로 상)에 존재하는 챔피언을 해당 진영의 미니언으로 인식하는 방법 적용
    - 아래는 각 진영별 챔피언을 미니맵 상에 표시하고 미니언을 표시한 결과와 병합하는 코드
    
    ```python
    def get_champ_map(champ_csv_row, minimap_size, query_size=19):
        blue_champ_map = np.zeros(minimap_size, dtype=bool)
        red_champ_map = np.zeros(minimap_size, dtype=bool)
        
        for i in range(10):
            x, y = champ_csv_row['%dp_x_coord'%(i+1)], champ_csv_row['%dp_y_coord'%(i+1)]
            
            if not np.isnan(x):
                x, y = int(x-query_size//2), int(y-query_size//2)
    
                if i < 5:
                    blue_champ_map[y:y+query_size, x:x+query_size] = True
                else:
                    red_champ_map[y:y+query_size, x:x+query_size] = True
                
        return blue_champ_map, red_champ_map
    
    def get_red_minion_map(minimap, champ_map, line_map):
      # Red minion
      red_minion_min_LAB = [60, 170, 150]
      red_minion_max_LAB = [125, 190, 165]
    
      red_minion_map = np.all(red_minion_min_LAB < cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB), 2) & np.all(cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB) < red_minion_max_LAB, 2)
      red_minion_map = red_minion_map & line_map
    
      champ_in_line_map = champ_map & line_map
      red_minion_map = red_minion_map | champ_in_line_map
    
      return red_minion_map
      
    def get_blue_minion_map(minimap, champ_map, line_map):
      # Blue minion
      blue_minion_min_LAB = [80, 110, 90]
      blue_minion_max_LAB = [140, 120, 110]
    
      blue_minion_map = np.all(blue_minion_min_LAB < cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB), 2) & np.all(cv2.cvtColor(minimap, cv2.COLOR_BGR2LAB) < blue_minion_max_LAB, 2)
      blue_minion_map = blue_minion_map & line_map
    
      champ_in_line_map = champ_map & line_map
      blue_minion_map = blue_minion_map | champ_in_line_map
    
      return blue_minion_map
    ```
    
    ![Daily%20Report%202021%2007%2001%20(Thu)%20fa2b172c2d5c4093836d48a4b7fbcfbf/ezgif.com-gif-maker_(9).gif](/images/0701_9.gif)
    
    - 위의 시각화 결과를 보면 단순히 챔피언을 미니언으로 인식하였기에 문제가 발생하는 경우가 있음. 예를 들어, 미드 라인에 주목하면 잭스가 로밍을 온 다음 블루팀 진영의 웨이브가 올 때 원이 비정상적인 위치 (블루팀 웨이브의 머리 부분)에 잡히는 부작용이 발생. 내일은 해당 원인을 명확히 분석한 다음 해결하는 방향으로 진행할 예정.
