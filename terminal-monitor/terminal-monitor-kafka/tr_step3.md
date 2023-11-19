## Test

Yeni bir terminal açıp uygulamamız çalışıyor mu test edelim.

Öncelikle, terminalden kullanıcının hangi adımda olduğunu anlamanın bir yöntemi olmadığı için hangi adımda olduğumuzu belirtmemiz gerekiyor.

```sh
export current_step=step1
```

3 tane error çıkartacak rastgele komutlar çalıştırın. Rastgele harflere basıp enter tuşuna basabilirsiniz.

```sh
cat ~/workspace/tips.txt
```

ipucunun geldiğini göreceksiniz.

Aynısını step2 için yapalım.
```sh
export current_step=step2
```

3 tane rastgele error komutu çalıştırın.

```sh
cat ~/workspace/tips.txt
```

Son 2 satırda şöyle bir çıktı gördüyseniz işlem tamamdır:
```sh
try to use git merge on both branches  |  
research optional flags for git hash object command  |
```
