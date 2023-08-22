# project_eggs
## 📖 Gantt :fire:

```mermaid
gantt
    title A Gantt Diagram
    dateFormat  YYYY-MM-DD
    section AI
    AI 기술테스트  : a1, 2020-10-14, 10d
    가상 얼굴 학습 및 환경세팅  : 2020-10-14, 10d
    얼굴 인식 개선 및 적용 : after a1, 10d
    가상 얼굴 이미지 생성 및 분류 : after a1, 4d

    section Front-end
    와이어프레임     :a1,2020-10-14  , 10d
    react 학습 및 적용 : after a1,  10d
    사진 업로드 및 설정 기능 :after a1 , 10d

    section Back-end
    django 학습 및 적용 : a1,2020-10-14 , 10d
    회원기능      :a2,after a1 , 10d
    친구기능      :after a1  ,10d
    결과 이미지저장,공유  : a3,after a2, 2d
    스티커 기능  : a4,after a3, 2d
```