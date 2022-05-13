# Thread-driven Server

===========================

# Thread

- 프로세스는 단 하나의 File Descriptor를 가진다. 따라서 Thread끼리는 File Descriptor를 공유한다.connfd를 메인 쓰레드에서 피어쓰레드로 넘겨줬으면 둘 중 하나(보통 클라이언트 서비스를 마친 피어 쓰레드)만 connfd를 닫아주면 된다.

# Producer and Consumer Problem

## 쓰레드 풀링

- __쓰레드를 미리 만들어 놓는다.__ 

쓰레드 풀링 서버에서는 쓰레드가 ```connect Request``` 가 ```Accept``` 될 때마다 생성되는 것이 아니라, 미리 만들어 놓고 _block_ 시켜놓고 있다가 ```connect``` 가 이루어지면 하나씩 깨워 쓰는 것이다. 


- ___main thread_ 는 _Producer_, _peer thread_ 는 _Consumer___

쓰레드 풀링은 기본적으로 Producer & Consumer Problem 이다.  
client의 request를 Accept하는 main Thread가 버퍼에 connfd를 저장(생산)하는 Producer이고, 버퍼로부터 connfd를 뽑아서 각 클라이언트와 통신하는 peer thread가 Consumer이다.
producer-consumer 문제에는 기본적으로 3개의 세마포어가 필요하다.

1. mutex: 버퍼에 대한 접근을 락하는 세마포어   
   
2. slots: 버퍼의 잔여 슬롯 수   
   Consumer 쓰레드에서 증가하고(버퍼를 하나 비움으로써), slots이 0이 되면 Producer 쓰레드가 block 된다.   
      
3. items: 버퍼에 저장된 아이템 수   
   Producer 함수에서 증가하고(V), items 가 0이 되면 Consumer 쓰레드는 block된다.

```C
void sbuf_insert(sbuf_t *sp, int item){
    P(&sp->slots); // 슬롯이 더 이상 남아 있지 않을 떄 다른 Producer Thread 를 블록하기 위해서
    P(&sp->mutex); // 다른 Threads 들의 버퍼에 대한 접근을 막기 위해서. Consumer Thread의 접근은 허용해도 되지 않나 싶은데...

    sp->buf[(++rear) % (sp->n)] = item;

    V(&sp->mutex);
    V(&sp->items);
}

```

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
   
# 
#
### Thread 함수 

```C
void *thread(void *vargp)
{
    Pthread_detach(pthread_self()); 
    while(1){
        int connfd = sbuf_remove(&sbuf);
        echo_cnt(connfd);
        Close(connfd); 
    }
}
```

무한루프를 돌고 있는 이유는 하나의 요청을 처리한 이후에도 쓰레드는 죽지 않고 다음 Connect를 기다리기 때문이다. 따라서 connfd를 버퍼에서 제거하는 문장을 while문 바깥에서 처리하면 안된다. 

# 
#



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

        /*7
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

### sbuf Package

# Semaphore 관련 함수

    Sem_init(sem_t sem, int pshared, unsigned int value)

- semaphore _sem_ 의 값을 value로 초기화시켜주는 함수.
- pshared 값이 0이면 세마포어를 쓰레드끼리 공유한다는 뜻이다. 세마포어가 global scope으로 선언되는 등 세마포어에 모든 쓰레드가 접근가능해야 한다.
- pshared 값이 nonzero면 세마포어를 **프로세스**끼리 공유한다는 뜻이다. 세마포어가 shared memory region에 위치해 있어야 한다.
  \_(자식 프로세스도 부모 프로세스의 메모리 매핑을 상속받으므로, 세마포어에 접근가능하다고 되어 있다)

(https://man7.org/linux/man-pages/man3/sem_init.3.html)
```
