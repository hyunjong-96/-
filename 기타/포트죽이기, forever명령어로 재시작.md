cmd에서 포트 죽이는법

1. cmd netstat -ano
2.  0,0,0,0 뒤의 포트의 pid확인
3. taskkill /f /pid /pid

forever list로 돌아가는 프로세스 확인

1. forever stop [index]번호로 죽인다
2. netstat -tnpl로 돌아가는 프로세스 확인
3. kill -9 [pid]로 프로세스 죽인다.
4. forever start -c "yarn start" ./ 실행