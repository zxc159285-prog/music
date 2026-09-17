# 무한 BGM 플레이어 트러블슈팅 아카이빙

## 1. Web Audio API 장시간 재생 시 오디오 지지직거림 및 멈춤 현상 (Audio Glitch & Stutter)

- **발생 일자**: 2026-09-18
- **해당 기능**: 무한 피아노/앰비언트 실시간 연주 엔진 (`playPianoNote`, `playAmbientPadChord`)
- **문제 현상**: 
  - 음악을 켠 후 일정 시간(수 분 이상)이 경과하면 소리가 정상적으로 나오지 않고 "지지직"거리는 디지털 클리핑 노이즈가 발생하며 음악이 끊기거나 멈춤.
- **원인 분석**:
  1. **오디오 노드 가비지 컬렉션(GC) 누수**: 매 박자마다 피아노 타건을 위해 수십 개의 `OscillatorNode`, `GainNode`, `BiquadFilterNode`를 동적 생성하였으나, 연주가 끝난 후 `node.disconnect()`를 명시적으로 호출하지 않아 수천 개의 오디오 노드가 브라우저 AudioContext 그래프에 그대로 살아있었음.
  2. **ConvolverNode(합성 리버브)의 극심한 CPU 과부하**: 실시간 임펄스 응답 연산에 수천 개의 누적 노드가 중첩 입력되면서 브라우저 오디오 스레드에 버퍼 언더런(Buffer Underrun)이 발생함.
- **해결 방안**:
  1. **노드 자동 청소기 도입**: 타건 함수 내부에서 생성된 모든 노드 배열(`activeNodes`)을 기록하고, 타건 지속시간(`duration + 0.2초`)이 만료되는 즉시 `node.disconnect()`를 실행하여 메모리와 오디오 그래프를 즉각 반환.
  2. **초경량 공간계 딜레이-피드백 리버브 구조 전환**: 무거운 Convolver 대신 `DelayNode(380ms)` + `Lowpass BiquadFilter(1600Hz)` + `Feedback Gain(0.32)`의 경량 순환 루프로 교체하여, 장시간 수십 시간 연속 재생에도 CPU 점유율을 0%대로 안정화함.
