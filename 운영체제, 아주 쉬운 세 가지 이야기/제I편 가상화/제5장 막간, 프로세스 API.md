# 대표적인 프로세스 API

- UNIX에서 사용하는 대표적인 시스템 콜로 fork(), wait(), exec()가 있음

## fork()

- 프로세스 생성에 사용되는 시스템 콜
- fork를 사용하는 코드의 예
  ```
  int main (int argc, char *argv[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if(rc < 0) {
      fprintf(stderr, "fork failed\n");
      exit(1);
    } else if(rc == 0) {
      printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
      printf("hello, I am parent of %d (pid:%d)\n", rc, (int) getpid());
    }
    return 0;
  }
  ```
- 위 코드는 다음과 같이 동작
  1. "hello world"와 PID(프로세스 식별자)를 출력
  2. fork() 시스템 콜을 통해 프로세스를 생성
  3. 자신이 부모 프로세스인지 자식 프로세스인지 출력
- 위 코드를 실행하면 다음과 같은 결과를 얻을 수 있음
  ```
  prompt ./p1
  hello world (pid:29146)
  hello, I am parent of 29147 (pid: 29146)
  hello, I am child (pid:29147)
  ```
- 결과를 통해 다음과 같은 정보를 알 수 있음
  - 첫 printf문은 한 번만 실행됨
    - fork()로 생성된 프로세스는 main() 함수의 처음부터 시작하지 않고 fork() 호출 이후부터 작업을 수행
  - 부모 프로세스와 자식 프로세스의 fork() 반환 값이 다름
    - 부모 프로세스는 생성된 자식 프로세스의 PID를, 자식 프로세스는 0을 반환받음
    - 이 외에도 자식 프로세스는 자신만의 주소 공간, 레지스터 등을 가짐
  - 부모 프로세스와 자식 프로세스의 실행 순서는 바뀔 수 있음
    - CPU 스케줄러가 상황에 맞는 프로세스를 실행하기 때문
    - 실행 순서를 알 수 없기 때문에 다양한 문제(ex. 병행성 문제)가 발생할 수 있음

## wait()

- 부모 프로세스가 자식 프로세스의 종료를 대기할 때 사용하는 시스템 콜
- wait() 대신 waitpid()도 사용할 수 있음
- wait() 시스템 콜을 사용한 코드의 예
  ```
  int main (int argc, char *argv[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if(rc < 0) {
      fprintf(stderr, "fork failed\n");
      exit(1);
    } else if(rc == 0) {
      printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
      int wc = wait(NULL);
      printf("hello, I am parent of %d (wc:%d) (pid:%d)\n", rc, wc, (int) getpid());
    }
    return 0;
  }
  ```
- 위 코드는 이전에 사용한 fork() 시스템 콜 코드에서 부모 프로세스 동작에 wait()을 추가한 것
  - 거의 동일하게 동작하지만 부모 프로세스가 자식 프로세스의 종료를 기다리기 때문에 출력문이 어떤 순서로 동작할 지 알 수 있음

## exec()

- fork() 시스템 콜이 자신의 복사본을 생성하여 실행한다면, exec()는 자신이 아닌 다른 프로세스를 실행할 때 사용
- exec()는 execl(), execle(), execlp(), execv(), execvp(), execve()의 6가지 변형이 존재
- exec() 시스템 콜을 사용한 코드의 예
  ```
  int main (int argc, char *argv[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if(rc < 0) {
      fprintf(stderr, "fork failed\n");
      exit(1);
    } else if(rc == 0) {
      printf("hello, I am child (pid:%d)\n", (int) getpid());
      char *myargs[3];
      myargs[0] = strdup("wc");
      myargs[1] = strdup("p3.c");
      myargs[2] = NULL;
      execvp(myargs[0], myargs);
      printf("this shouldn't print out");
    } else {
      int wc = wait(NULL);
      printf("hello, I am parent of %d (wc:%d) (pid:%d)\n", rc, wc, (int) getpid());
    }
    return 0;
  }
  ```
- 위 코드는 이전에 사용한 wait() 시스템 콜 코드에서 자식 프로세스 동작에 wc 명령어를 exec()로 실행하는 코드를 추가한 것
  - 부모 프로세스의 동작은 이전과 동일
  - 자식 프로세스는 hello...문을 출력한 뒤 인자를 설정하고 exec() 시스템 콜을 통해 다른 프로그램을 호출
    - exec()를 통해 호출한 프로그램의 코드와 정적 데이터를 읽어 들여 현재 실행 중인 프로세스의 코드와 정적 데이터 부분을 덮어 씌움
    - 힙, 스택 등의 프로그램이 사용하는 주소 공간들도 새로운 프로그램 실행을 위해 다시 초기화 됨
    - 즉, exec()는 새로운 프로세스를 만드는 것이 아닌 기존의 프로세스를 덮어 씌우는 방식으로 동작
    - 이런 이유로 exec() 아래에 있는 출력문을 수행되지 않음

# 이런 API를 사용하는 이유

- UNIX 쉘을 구현하기 위함
- 쉘에서 명령어를 실행하면 fork()로 새 프로세스를 생성하고 exec()로 코드를 덮어 씌우는 방식으로 동작
  - 이런 방식으로 구현하면 프로세스를 다룰 때 유용함
  - 예를 들어, 표준 입출력을 바꿀 일이 있다면 exec() 시스템 콜을 호출하기 전에 표준 입출력을 변경하기만 하면 됨
- 이 외에도 이 API들에 대해 알아야 할 것이 많지만 이는 뒤에서 다룰 예정

# 그 외의 API들

- kill: 프로세스에게 시그널(signal)을 보낼 때 사용
- ps: 어떤 프로세스가 실행 중인지 확인할 때 사용
- top: 시스템에 존재하는 프로세스와 각 프로세스가 자원들을 얼마나 사용하지는 확인할 때 사용

# 요약

- 가장 기본적인 시스템 콜에 대해서만 다뤘기 때문에 더 많은 내용을 따로 공부하는 것을 추천
