# 셸 만들기

## **셸**이란?

**명령어와 프로그램**을 **실행**할 때 사용하는 **인터페이스**이다!

자세히 말하자면 셸은 **커널과 사용자간의 다리** 역할을 하는 것으로 사용자로부터 명령을 받아 그것을 해석하고 프로그램을 실행하는 역할을 한다!

## 셸의 주요 기능

1. 사용자와 커널 사이에서 명령을 해석해 전달하는 **명령어 해석기 기능**
2. 셸 자체의 프로그래밍 기능으로 프로그램을 작성할 수 있는 **셸 스크립트**
3. **사용자 환경 설정의 기능**

## 셸의 종류

### Bourne Shell 호환

- 본 셸 (sh)
- 암키스트 셸 (ash)

    데비안 암키스트 셸(dash)

- Bash

    콘 셸 (ksh)

    Z 셸 (zsh)

### C shell 호환

- C 셸 (csh)

    테넥스 C 셸 (tcsh)

# 코드 분석

우선 깃허브의 brenns10 이라는 유저가 만든 lsh를 뜯어보고 분석한 후

나만의 셸을 만들기로 한다

[https://github.com/brenns10/lsh](https://github.com/brenns10/lsh) - lsh 코드

## 생명주기

일반적인 셸은 아래와 같은 생명주기를 가지고 있다

1. **Initialize (초기화)** :

    셸 초기화 파일을 불러와 셸의 환경을 설정한다

2. **Interpret (해석 및 실행)** :

    사용자가 입력한 명령어를 해석하고 이를 실행한다

3. **Terminate (종료)** :

    명령어가 종료되고, 셸이 메모리를 청소하는 과정이다

## main()

```c
int main(int argc, char **argv)
{
  // 컨픽 파일 불러오기

  // 명령어 받기 반복부
  lsh_loop();

  // shutdown/cleanUp 코드 작동부

  return EXIT_SUCCESS;
}
```

lsh는 간단한 셸이기 때문에 초기화 과정과 종료 과정은 포함하고 있지 않다.

## lsh_loop()

```c
void lsh_loop(void)
{
  char *line;
  char **args;
  int status;

  do {
    printf("> ");
    line = lsh_read_line(); // 명령 받기
    args = lsh_split_line(line); // 명령 분리
    status = lsh_execute(args); // 명령 실행

    free(line); // 메모리 해제
    free(args); // 메모리 해제
  } while (status);
}
```

Interpret 과정의 코드는 exit 명령이 실행되기 전까지 명령어 받는 과정을 반복하며, 명령어는 lsh_read_line()로 받고lsh_split_line()로 명령을 분리해준 후 lsh_execute()로실행한다.

## lsh_read_line()

```c
#define LSH_RL_BUFSIZE 1024
char *lsh_read_line(void)
{
  int bufsize = LSH_RL_BUFSIZE;
  int position = 0;
  char *buffer = malloc(sizeof(char) * bufsize);
  int c;

  if (!buffer) {
    fprintf(stderr, "lsh: allocation error\n");
    exit(EXIT_FAILURE);
  }

  while (1) {
    // 글자 받음
    c = getchar();

    // EOF(^Z / ^C 등) 이나 엔터입력(\n)을 받으면 해당 위치에 null(\0) 대입후 반환
    if (c == EOF || c == '\n') {
      buffer[position] = '\0';
      return buffer;
    } else {
      buffer[position] = c;
    }
    position++;

    // 버퍼 크기보다 큰 입력값을 받으면 버퍼의 크기를 재할당해준다
    if (position >= bufsize) {
      bufsize += LSH_RL_BUFSIZE;
      buffer = realloc(buffer, bufsize);
      if (!buffer) {
        fprintf(stderr, "lsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }
  }
}
```

유저 입력 받는 과정을 간단히 getline()을 이용해서 처리할 수 있겠으나, brenns10은 getline이 없는 전통적인 input처리 코드를 공부해보라는 목적으로 위와 같은 코드를 작성했다고 한다.

```c
char *lsh_read_line(void)
{
  char *line = NULL;
  ssize_t bufsize = 0; // have getline allocate a buffer for us

  if (getline(&line, &bufsize, stdin) == -1){
    if (feof(stdin)) {
      exit(EXIT_SUCCESS);  // We recieved an EOF
    } else  {
      perror("readline");
      exit(EXIT_FAILURE);
    }
  }

  return line;
}
```

이 코드는 brenns10이 작성한 getline()을 사용한 lsh_read_line()이다

## lsh_split_line()

```c
#define LSH_TOK_BUFSIZE 64
#define LSH_TOK_DELIM " \t\r\n\a" // tab, Carriage return, New line, Alert
char **lsh_split_line(char *line)
{
  int bufsize = LSH_TOK_BUFSIZE, position = 0;
  char **tokens = malloc(bufsize * sizeof(char*));
  char *token;

  if (!tokens) {
    fprintf(stderr, "lsh: allocation error\n");
    exit(EXIT_FAILURE);
  }

	//입력받은 line을 " \t\r\n\a" 로 나눠준다
  token = strtok(line, LSH_TOK_DELIM);
  while (token != NULL) {
    tokens[position] = token;
    position++;

		// 버퍼 크기보다 큰 입력값을 받으면 버퍼의 크기를 재할당해준다
    if (position >= bufsize) {
      bufsize += LSH_TOK_BUFSIZE;
      tokens = realloc(tokens, bufsize * sizeof(char*));
      if (!tokens) {
        fprintf(stderr, "lsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }

    token = strtok(NULL, LSH_TOK_DELIM);
  }
  tokens[position] = NULL;
  return tokens;
}
```

## lsh_launch()

```c
int lsh_launch(char **args)
{
  pid_t pid, wpid;
  int status;

  pid = fork(); //현재 부모 프로세스를 복제(Git의 fork와 비슷함)
  if (pid == 0) {
    // 복제된 자식 프로세스
    if (execvp(args[0], args) == -1) {
      perror("lsh");
    }
    exit(EXIT_FAILURE); // execvp로 실행한 프로그램이 종료되면, 자식 프로세스를 종료한다.
  } else if (pid < 0) {
    // fork 중 에러 생김
    perror("lsh");
  } else {
    // 부모 프로세스
    do {
      wpid = waitpid(pid, &status, WUNTRACED);
    } while (!WIFEXITED(status) && !WIFSIGNALED(status));
  }

  return 1;
}
```

쉘이 명령을 실행하는 과정은 다음과 같다

1. 부모 프로세스(현재 프로세스)를 복제한다, 이때 복제된 프로세스를 자식 프로세스라 부른다
2. 유저의 입력값을 파싱해서 얻어낸 프로그램 이름을 환경변수에서 찾는다 (PATH)
3. 유저가 입력한 프로그램이 환경 변수에 존재하면, 이 프로그램을 자식 프로세스에서 실행한다
4. 이때 부모 프로세스는 자식 프로세스가 종료될때까지 대기한다
5. 자식 프로세스가 종료되면 부모 프로세스는 계속 진행한다

---

참고한 글 :

[https://brennan.io/2015/01/16/write-a-shell-in-c/](https://brennan.io/2015/01/16/write-a-shell-in-c/)

[https://jhnyang.tistory.com/57](https://jhnyang.tistory.com/57)
