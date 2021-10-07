---
layout: post
title:  "2021-06-30 Code refactoring"
date:   2021-06-30 16:40:00
categories: gamequant

---

1. **Code Refactoring**
- 기존 코드가 jupyter notebook에 너무 난잡하게 짜여져 있어 code refactoring 진행
- 정리한 코드 구조
- main.py : main code
- utils
└  io.py : input/output
└  champ_tracking.py : functions for champion icon tracking 
└  jg_tracking.py : functions for jungle monster tracking 
└  minion_tracking.py : functions for minion tracking 
└  time_log.py : functions for kill and object log tracking
- 정리된 main.py 코드

    ![Daily%20Report%202021%2006%2030%20(Wed)%207d7e3a2629e444499b2e2ae86c1ca1d2/Untitled.png](/images/0630.png)
    
2. **Champion icon tracking**
    - 성능 향상보다는 속도 향상 위주로 진행.
    - 기존의 코드로 진행할 경우, 1초에 약 1~2프레임밖에 처리를 하지 못하였기 때문에 약 50초의 영상을 처리하기 위하여 20분 가량의 시간 소요
    - 속도 향상을 위하여 일정 간격 마다의 프레임만 예측하고 그 사이 프레임은 interpolation을 통하여 채워넣는 방식 적용
    - Default 값을 10으로 설정하였으므로 10개의 프레임 간격으로 예측
      
        ```python
        parser.add_argument('--skip_interval', default=10, type=int, help='skip some frames to speed up the process')
        
        time_step = 0
        for frame in frames:
            if time_step % skip_interval == 0:
        			# DO WORK
        		else:
        			# SKIP
            
            time_step += 1
        ```
        
    - 비어있는 프레임을 단순한 linear interpolation을 통하여 채워넣고자 하였으나 귀환 혹은 텔레포트 사용의 경우, 위치가 짧은 프레임 사이에 굉장히 많이 이동하여 문제가 발생.
    - 이를 해결하기 위하여 연속된 프레임 사이의 좌표값 변화가 기준치(delta_thrs=80)보다 큰 경우, 해당 프레임 전과 후를 별개의 프레임으로 분리한 다음 각각에 대해 따로 interpolation을 진행하여 비정상적으로 interpolation 되는 현상을 해결.
    
    - 문제가 발생한 영상과 해결된 영상 (세나와 럼블의 귀환에 주목)
      
        ![Daily%20Report%202021%2006%2030%20(Wed)%207d7e3a2629e444499b2e2ae86c1ca1d2/ezgif.com-gif-maker_(2).gif](/images/0630_gif_2.gif)
        
        ![Daily%20Report%202021%2006%2030%20(Wed)%207d7e3a2629e444499b2e2ae86c1ca1d2/ezgif.com-gif-maker_(3).gif](/images/0630_gif_3.gif)
        
        ```python
        def interpolate_coord(coord_x, coord_y, delta_thrs=80):
          '''
          if the delta value (next position - current position) is bigger than delta_thrs,
          it is considered a point where there has been a sudden change in location. (e.g., teleport)
          And the smoothing process is adopted for each interval.
          '''
          notna_idcs = coord_x.notna()
        
          x_notna = coord_x[notna_idcs].values
          y_notna = coord_y[notna_idcs].values
            
          coord_delta = np.sqrt((x_notna[1:] - x_notna[:-1])**2
                                + (y_notna[1:] - y_notna[:-1])**2)
        
          split_idcs = np.where(coord_delta > delta_thrs)[0]+1
          frame_idcs = coord_x[coord_x.notna()].index[split_idcs]
            
          if len(frame_idcs) > 0:
            coord_x_split = np.split(coord_x.values, frame_idcs)
            coord_y_split = np.split(coord_y.values, frame_idcs)
          else:
            coord_x_split = [coord_x.values]
            coord_y_split = [coord_y.values]
          
          coord_x_itp = []
          coord_y_itp = []
          for coord_x_part, coord_y_part in zip(coord_x_split, coord_y_split):
            coord_x_itp.append(pd.DataFrame(coord_x_part).interpolate().values)
            coord_y_itp.append(pd.DataFrame(coord_y_part).interpolate().values)
            
          coord_x_itp = np.concatenate(coord_x_itp)[:,0]
          coord_y_itp = np.concatenate(coord_y_itp)[:,0]
            
          return coord_x_itp, coord_y_itp
        ```
        
    - 또한 기존에 query image를 다채롭게 테스트하여 성능을 높이고자 하였는데, 속도 대비 성능 향상 폭이 적을 뿐만 아니라 노이즈를 발생시킬 가능성이 큰 것을 확인하여 **hyper-parameter grid 크기 감소**
    - crop_pxs: 9,10,11,12 (기존) → 9,11 (변경)
    - top_k: 5 (기존) → 2 (변경)
    - 위와 같은 과정을 적용하여 기존에 1초당 1~2 프레임을 수행하던 알고리즘에서 1초당 35~40 프레임을 수행할 수 있도록 속도가 향상되어 약 25배 가량 빨라짐 🙂 (e.g., 20초 영상에 대해 약 24초의 시간 소요)
    - 예시 결과 시각화
      
        ![Daily%20Report%202021%2006%2030%20(Wed)%207d7e3a2629e444499b2e2ae86c1ca1d2/ezgif.com-gif-maker_(4).gif](/images/0630_gif_4.gif)
    
3. Code Refactoring 작업으로 인하여 **minion tracking 성능 향상**은 진행하지 못함.
- 아이디어: (챔피언 아이콘 예측과 같이) 예측된 교착 지점을 바탕으로 bspline에 fitting 시킨 후 중간 지점을 interpolation하여 채우는 방식
