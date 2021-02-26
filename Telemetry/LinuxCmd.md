# awk
```shell
awk '{print $1 $2 $4}' ./filename # 구분자 기준으로 1,2,4번 출력 (-F로 구분자 지정, default ' ')
```
# sed
## all quote -> double quote
```bash
sed "s/'/\"/g
```
# sort
- 정렬
```shell
cat file | sort
```
# uniq
- 유니크 값만(정렬된 상태에서만 사용)
```shell
cat file | sort | uniq
```

awk -F , '{ print $3 $9 $22 }' ingress-nginx-controller-8d5k7_ingress-nginx_ingress-nginx-controller-c9dea527676af42d5cdb9de80e4172d66e79059a447616d7efff9047325558e8.log | grep kemi-api

# ln
- Link file 생성
## Symbolic Link
- 원본 파일을 가리키도록 링크만 시켜둔 것
- Window에서 바로가기 같은 것(크기와는 무관하다)
```bash
ln -s /tmp /var/tmp
```
## Hard Link
- 원본 파일과 동일한 내용의 다른 파일
- 하나를 삭제하더라도 하나는 남아 있음
- 원본 파일이 변경되면 링크파일 내용도 자동 변경
```bash
ln hard_source hard_link
```
# tmux
- terminal multiplexer
- ssh로만 사용시 접속이 끊길 수도 있으므로 위험
- 한 세션에 여러 명이 함께 볼 수 있음
- 세션에서 실행 후 나갔다가 다시 세션으로 들어와도 기존 이력을 볼 수 있음
## 구성
| 명칭    | 설명        |
|-------|------------|
|session | tmux 실행 단위, 여러개의 window로 구성|
|window | terminal 화면, 세션 내에서 여러 탭처럼 사용 가능|
|pane |하나의 window 내에서 화면 분할|
|status bar | 화면 아래 표시되는 막대|
## 명령어
- tmux prefix : ctrl + b
### 세션 관련
```bash
$ tmux new -s <session-name> # 새 세션 생성
ctrl+b, $ #세션 이름 수정
$ exit # session 종료
ctrl+ b,d # session 중단(detach)
$ tmux ls # session 목록 보기(list-session)
$ tmux attach -t <session_name | session_number> # 세션에 attach
```
### 윈도우 관련
- ctrl +b, c # 새로운 윈도우 생성
- $ tmux new -s <session-name> -n <window-name> # 세션 + 윈도우 생성
- ctrl + b, , # 윈도우 이름 수정
- ctrl + b, & # 윈도우 종료
- ctrl + d
- ctrl + b, 0-9,n,p,l,w,f # 윈도우 이동


### Pane 관련
- ctrl + b, % : 횡 분할
- ctrl + b, " : 종 분할
- ctrl + b, x : Pane 삭제

