# 작성 목적
   파이프라인화된 드라이버의 트랜잭션 처리 및 전송 로직 및 콜백 관련 단원 학습 후 배운 내용을 병합한 코드 작성 연습
   
#  사용한 UVM feature
   a. uvm_callback 사용 메커니즘
   test 클래스에서 `uvm_callbacks(my_driver, driver_cb)::add(my_drv, cb_inst)를 실행했다고 가정하고 작성했음
   드라이버 클래스 ``uvm_register_cb(my_driver, driver_cb) 매크로를 이용해 콜백을 등록 후
   uvm_do_callbacks를 이용해 콜백이 실행되도록 함
   
   b. seq_item_port를 이용한 드라이버-시퀀서 상호작용(handshake)
   여러 개의 트랜잭션을 보내게 될 경우 시퀀서가 드라이버에서 처리가 완료된 트랜잭션을 식별 가능하게 하기 위해
   rsp를 사용해 req를 cast해주고 set_id_info를 통해 트랜잭션 id, 시퀀스 id를 복사해 item_done의 인자로 rsp를 넣어줬을 때 시퀀서가
   정확한 트랜잭션을 식별 가능하도록 해주었음

   c. 설정 메커니즘을 이용헤 connect_phase에서 가상 인터페이스 연결
   top module의 initial 블록에서 인터페이스를 set 해주었다고 가정했음
   인터페이스는 클래스 내부에서 인스턴스화할 수 없으므로 가상 인터페이스를 사용했으며 if문과 `uvm_error 메시지 매크로를 이용해 가상 인터페이스를 얻지 못할 경우
   디버깅 메시지를 확인 가능하도록 만들었음
   
# 발생했던 문제1
   코드 실행 결과 트랜잭션이 빠르게 연속적으로 보내짐에 따라 인터페이스 신호가 오염(레이스 컨디션)될 수 있음을 발견했음
   
# 해결방법
   이 문제를 해결하기 위한 방법을 찾던 주 세마포를 이용한 방식을 찾아낼 수 있었음 세마포를 이용해 dut로 트랜잭션 신호가 구동되는 동안 다른 트랜잭션 신호가
   드라이버로 구동되는 일이 발생하지 않도록 send_to_dut 직전에 semaphore의 key가 get되도록 했으며 리셋 메서드 또한 문제가 생길 수 있을 것이라 판단해 put은
   reset메서드 이후에 실행되도록 했음.




# 발생했던 문제2
   reset 메서드를 한 개만 사용할 경우 페이즈 병렬화 이용 시 signal ownership이 명확하지 않아 문제가 생길 수 있음을 알 수 있었음
   
# 해결방법
   3-2의 문제의 경우 리셋 메서드를 하나만 정의할 경우 이 메서드에서 data, addr 관련 신호를 모든 초기화하므로 데이터부와 주소부로 나뉘는 페이즈 병렬화의 취지에
   맞지 않아 생기는 문제가 있음을 알게 되었음. 이를 수정하기 위한 방법으로 리셋 메서드 분해,
   또는 send_to_dut 메서드 분해를 찾을 수 있었음(이 경우 run_phase 아래에 메서드를 만들어 트랜잭션을 받아 이 메서드의 아래에 있는 두 개의 메서드에 트랜잭션을
   인자로 넣어줌)
   이번 코드 작성에서는 첫 번째 방법을 이용해 해결할 수 있었음
   
# 배운 점
   이번 코드 작성을 통해 개인적으로 아래의 내용들을 배우게 되었음
   1. get에서 get_full_name() 대신 " " 사용도 가능함
   2. multiple outstanding transaction 환경에서 세마포를 이용해 인터페이스 신호 오염 방지
   3. 클럭킹 블록을 이용해 dut와 TB 사이 레이스 컨디션 줄여주기(클럭킹 블록은 샘플링, 드라이빙 시 스큐를 제공해줌)
   4. 페이즈 병렬화 시 데이터부 제어부 사이 신호 소유권을 명확히 하기
   => a. 태스크를 따로 주거나(이 경우 트랜잭션을 공통으로 받기 위해 run - drive_transfer-addr/data)로 구분해줌
   => b. 리셋 메서드를 따로따로 만들기
   5. 신호 초기화 시 z보다는 0으로 바꿀 것을 추천함
   6. send_to_dut를 전후로 세마포 get, put 실행(put은 리셋 메서드 이후에)
   7, `uvm_do_callbacks 사용 시 메서드 인자는 공통되게 설정
   8. multiple outstanding transaction 환경에서 set_id_info, automatic 등을 사용
   9. 페이즈 병렬화의 개념
  
   또한 추가적으로 ai를 이용해 다음의 내용들을 배울 수 있었음
   10. fork/join_none 사용시 스레드 수명 주의
   부모 태스크가 먼저 끝나서 지역 변수 수명, 트랜잭션 핸들 공유, dangling 프로세스 문제 발생 가능
   => automatic, transaction clone/copy, wait fork등 고려 필요

   11 queue 접근도 서로 다른 스레드가 push, pop 접근(producer-consumer 구조)
   => 메일박스, tlm_fifo, 세마포 보호를 더 많이 사용함 => UVM의 경우 uvm_tlm_analysis_fifo, uvm_tlm_fifo를 많이 사용함
   
   12. wait(req_list.size() > 0) busy-wait 가능성 => 
   단순하나 이벤트 구동 동기화보다 비효율적일 수 있음 -> event, 메일박스 블로킹 get, fifo get 방식이 더 깔끔함

   14. APB protocol timing 명확화
   APB는 보통 셋업, 인에이블 페이즈 2사이클 구조 => 파이프라인 구조를 만들어도
   addr valid timing, ready sampling timing을 프로토콜 spec 기준으로 맞추는 게 중요 => addr phase, data phase를 분리 했어도 타이밍 violation 여부 확인

   16. 콜백은 기능 삽입용 훅(pre_tx : 전송 전 수정/검사, post_tx: 후처리/커버리지/에러 주입)

   17. 클로킹 블럭 사용시 cb를 거쳐서 접근 ~~~~.cb.~~
   
   18 신호 cleanup, 프로토콜 completion은 다름(handshake 완료 != signal reset 완료)
   => cleanup 이후 unlock, transction done 처리가 안전

   19. 세마포 갯수 = 소유권 단위
   addr/data 독립 파이프라인 가능 -> 세마포 분리, 전체 bus atomic access 필요 -> 세마포 1개

   20. 리셋 태스크 내부 wait 위치 조심 => 마지막에 @(clk)로 한 번 대기하게 하기  => 다음 get과 현재 put이 겹쳐버리지 않게
