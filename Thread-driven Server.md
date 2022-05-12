# Thread-driven Server

===========================

# Thread

- 프로세스는 단 하나의 File Descriptor를 가진다. 따라서 Thread끼리는 File Descriptor를 공유한다.connfd를 메인 쓰레드에서 피어쓰레드로 넘겨줬으면 둘 중 하나(보통 클라이언트 서비스를 마친 피어 쓰레드)만 connfd를 닫아주면 된다.

# Producer and Consumer Problem

이렇게 하면 안되나 하는 의문점을 가지게 되었다.

```

// Consumer 입장
P(&mutex);

if(sp->items != 0){
    item = sp->buf[(++sp->front) % sp->n];
    sp->items --;
    sp->slots ++;
}

V(&mutex);
```

```
// Producer 입장
P(&mutex);

if(sp->slots != 0){
    sp->buf[(++sp->rear) % sp->n] = item;
    sp->slots ++;
    sp->items ++;
}
V(&mutex);
```

내가 서비스의 클라이언트 입장이라고 생각해보자.

어떤 item을 get하고자 하는 request 요청을 보냈는데, 잔여 item이 0이면 서버는 item을 반환하지 않고 요청을 return 해버린다.
이건 클라이언트로서 바라는 것은 아닐 것이다.
요컨대, concurrent client가 많아 내 요청을 바로 해결할 수 없는 경우에는 resource의 여유가 있을 때까지 잠시 블럭해놓고 요청에 대해 성공적으로 response를 날리는 것이 바람직한 것이지,
resource가 있을 때에만 response를 보내고 그렇지 않은 때에는 client가 요청한 바구니를 empty한 상태로 return 하면 안되는 것이다.

# Readers-Writers Problem

reader는 공유 resource의 값(상태)를 바꾸지 않기 때문에 critical sectino에 들어갈 때 락을 걸 필요가 없다.

그래서 아래의 예제에서도 mutex를 건 것은 현재 critical section에 진입한 reader의 수, 즉 readcnt 값을 변화시키기 위해서 락을 걸 뿐 critical section에 대해서는 락을 걸지 않는다. 첫번째 reader가 진입했을 때 writer에 대해 락을 걸고 일련의 reader 들에 대해 계속 입장을 허용하다가(reader 우선순위 > writer 우선순위), reader가 더 이상 존재하지 않을 때(readcnt == 0) 비로소 writer에 대한 락을 해제한다("writer야 들어가~")

```
void reader(void){
    while(1){ // iterative reading
        P(&mutex);
        readcnt ++;
        if(readcnt == 1) // 첫 reader 에 대해서만 w를 락 건다.
            P(&w);
        V(&mutex);

        /*
        critical section
        read happens
        */

        P(&mutex);

        readcnt -- ;
        if(readcnt == 0) // 더 이상 reader 가 없으면 w에 대한 락을 해제한다.
            V(&w);
        V(&mutex);
    }
}
```

# Semaphore 관련 함수

    Sem_init(sem_t sem, int pshared, unsigned int value)

- semaphore _sem_ 의 값을 value로 초기화시켜주는 함수.
- pshared 값이 0이면 세마포어를 쓰레드끼리 공유한다는 뜻이다. 세마포어가 global scope으로 선언되는 등 세마포어에 모든 쓰레드가 접근가능해야 한다.
- pshared 값이 nonzero면 세마포어를 **프로세스**끼리 공유한다는 뜻이다. 세마포어가 shared memory region에 위치해 있어야 한다.  
  \_(자식 프로세스도 부모 프로세스의 메모리 매핑을 상속받으므로, 세마포어에 접근가능하다고 되어 있다)

(https://man7.org/linux/man-pages/man3/sem_init.3.html)
