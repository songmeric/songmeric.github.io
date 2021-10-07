---
layout: post
title:  "Automated data collection pipeline via frame-by-frame SSIM analysis - 3"
date:   2021-06-24 16:40:00
categories: gamequant

---

1. 플레이어의 위치 예상 AI 모델 학습용 csv 데이터셋 확보를 위한 코드 제작
- **csv에 기록되는 값**: Time step, here (챔피언 ID; 해당 [링크](https://drive.google.com/file/d/1rXuFiyo76iY5U7slszldTIo5hl0QTb6y/view?usp=sharing)의 챔피언 ID 엑셀 파일 사용), x_coord, y_coord, position (탑:0, 정글:1, 미드:2, 원딜:3, 서폿:4)
- **JgPos v1.2**이 예측하지 못한 프레임의 경우, 별도의 방법(e.g., 전후의 프레임 정보 활용)을 이용하여 해당 프레임에 누락된 챔피언의 위치를 기입할 예정
- JgPos v1.2 기반 **샘플 csv 데이터셋** ([링크](https://drive.google.com/file/d/1--vd_Q-Pi_Y6OzAwr8w0y21HC7xAXDzX/view?usp=sharing))

    ![Daily%20Report%202021%2006%2024%20(Thu)%204b8d5618d651402fa8be585835aad7d4/_2021-06-24__10.57.31.png](/images/_2021-06-24__10.57.31.png)

- 단순히 누락된 프레임을 가장 가까운 앞뒤의 프레임에 존재하는 해당 챔피언의 위치 값의 평균 (linear interpolation)으로 대체한 결과 시각화
    - 루시안이 텔레포트를 사용하는 경우를 기존 모델에서는 잡아내지 못하였으나 보완된 모습
    - 하지만 False positive의 경우, 평균값으로 중간 프레임을 채우기 때문에 맵을 가로지르는 현상 발생.
    - **결론: 엉뚱한 좌표에 예측하는 false positive를 해결한 뒤에 csv interpolation을 적용하면 누락되는 경우를 보완할 수 있을 것으로 예상됨**.

    ![Daily%20Report%202021%2006%2024%20(Thu)%204b8d5618d651402fa8be585835aad7d4/ezgif.com-gif-maker.gif](/images/0624_1.gif)

- 사용된 코드

```jsx
time_step = 0
csv_rows = []

# For each frame
for frame in tqdm(frames[20:]):
  # csv row initilization
  csv_row = [time_step]

  if frame is not None:
    # Get minimap part from the entire frame
    minimap = frame[map_y_min:map_y_max, map_x_min:map_x_max]
    minimap_clone = minimap.copy()

    # For each icon which exist in the current game
    for name, (icon, position, ID) in icon_dict.items():
      matched = False
      crop_px = 0

      while not matched:
        # Center-cropping
        if crop_px > MAX_CENTER_CROP:
          break
        if crop_px > 0:
          icon_query = icon[crop_px:-crop_px, crop_px:-crop_px].copy()
        else:
          icon_query = icon.copy()

        # Resize query icon to specific size
        icon_mini = cv2.resize(icon_query, query_size)

        # Find the optimal position for the entire icon, and the top, bottom, left and right part
        for icon in [icon_mini, icon_mini[:CROP_PX], icon_mini[-CROP_PX:], icon_mini[:, :CROP_PX], icon_mini[:, -CROP_PX:]]:
          # Extract the dominant HSV color from the icon
          icon_HSV = cv2.cvtColor(cv2.resize(icon, (1,1)), cv2.COLOR_BGR2HSV).astype(np.float32)
          (tH, tW) = icon.shape[:2]

          # Find the matched region using the query icon as a template
          result = cv2.matchTemplate(minimap, icon, cv2.TM_CCOEFF_NORMED)
          scores = np.sort(result.reshape(-1))[::-1]

          # for top-k matched regions
          for score in scores[:top_k]:
            (yCoords, xCoords) = np.where(result == score)

            for x, y in zip(xCoords, yCoords):
              # matched region comparison (HSV difference score and SSIM)
              matched_part = minimap[y:y+tH, x:x+tW].copy()
              matched_HSV = cv2.cvtColor(cv2.resize(matched_part, (1,1)), cv2.COLOR_BGR2HSV).astype(np.float32)
              HSV_diff = np.mean(np.abs(icon_HSV - matched_HSV))
              ssim = structural_similarity(icon, matched_part, multichannel=True)

              # check the matched region is valid
              if (HSV_diff < 10 and ssim > 0.35) or (HSV_diff < 25 and ssim > 0.7) or (HSV_diff < 15 and ssim > 0.55):
                matched = True
                csv_row.extend([ID, x+tW//2, y+tH//2, position])
                break

            if matched:
              break
          if matched:
            break

        # Center-cropping for one more pixel
        crop_px += 1

      if not matched:
        csv_row.extend([ID, np.nan, np.nan, position])

    csv_rows.append(csv_row)
    time_step += 1

csv_df = pd.DataFrame(csv_rows, columns=csv_columns)
csv_df.to_csv('JgPos_v1.2_dataset.csv', index=False)
```
