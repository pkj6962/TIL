# May, 2022 TIL

# GitHub

## Problem Solving

> ### Difference between main and master branch

> ### Git Push Error

_Updates were rejected because the remote contains work that you do
not have locally. This is usually caused by another repository pushing
to the same ref. You may want to first integrate the remote changes
(e.g., 'git pull ...') before pushing again._

`git pull origin master`로 1차 문제 해결.  
pull한다고 원격 저장소 상 파일이 로컬로 추가되는 건 아니더라.(왜? 추가되는 거 아니었음?)

"error: src refspec master does not match any"

### Command

#### Basic Command

    git init
    git remote add origin <RepositoryAddress.git>
    git remote -v
    git status
    git add <Filename>
    git reset HEAD -- path/to/file
    git commit -m "message"
    git push

#### Branch Command

git switch <branchname> // 현재 브랜치 이동
git switch -v <branchname> // 새로운 브랜치 생성 후 이동
git merge <branchname> // 현재 브랜치에 인자 브랜치를 병합. 원격저장소에는 반영되지 않음.

\*브랜치는 사람 별로 X, 기능별로 O

#### Pull Request (PR)

"내가 수정한 코드를 가져다가 병합해주세요"

master 담당자가 merge하고 confirm 한다.

# Computer Architecture

# Network

# System Programming

## Stock server Project

### Event-based Server

### Thread-based Server

#### Thread 함수

    Pthread_join(tid, NULL) // peer thread를 reap하는 함수(종료될 때까지 기다림)

##### printf

### 성능 분석

# C/C++

####  Difference between void and void

void\* 반환형 함수는 return NULL이 맞는 문법이지만, void 반환형 함수는 그 경우 warning message를 내보낸다.

```
void* thread(void *vargp){
    long niters = *((long*)vargp);
    long i;

    for(i; i < niters; i++){
        cnt ++; // critical zone
    }
    return NULL;
}
```

#### strchr

    char* strchr(char *string, char key)
    string에서 원하는 문자의 위치를 찾아 포인터로 반환한다.

#

#### execve(char*, char* argv, int option)

두 번째 인자인 argv의 마지막 원소는 NULL이어야 한다. 그렇지 않으면 ```bad address``` 오류를 반환한다.  


#
#


# Etc

## Data Structure

### AVL Tree

AVL Tree는 좌우 자식 트리의 높이의 차가 1 이하인 balanced Tree를 의미한다. 이러한 균형을 유지하기 위해서 새로운 노드를 삽입하여 균형이 깨질 때마다 LL, LR, RL, 또는 RR Rotation을 수행해야 한다.

이때 수행의 주체가 되는 노드는 BalanceFactor가 0, 1, -1을 넘어서는 노드이고, 이를 피벗이라고 한다.

https://blog.naver.com/pjh6962/222600073342

## Github

## Vscode
