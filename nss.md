# Name Service Switch

Изначально источники данных были фиксированные, но возникла потребность динамически менять их без изменения кода программ. Так появился файл nsswitch.conf и динамические библиотеки для запроса разных источников. Спустя 20 лет этот функционал добавили в GNU libc.

```c
#include <stdio.h>
#include <sys/types.h>
#include <pwd.h>

int main() {
    struct passwd *p;
    uid_t  uid = 1000;

    if ((p = getpwuid(uid)) == 0)
        perror("getpwuid() error");
    else {
        printf("getpwuid() returned the following info for uid %d:\n", (int) uid);
        printf("  pw_name  : %s\n",       p->pw_name);
        printf("  pw_uid   : %d\n", (int) p->pw_uid);
        printf("  pw_gid   : %d\n", (int) p->pw_gid);
        printf("  pw_dir   : %s\n",       p->pw_dir);
        printf("  pw_shell : %s\n",       p->pw_shell);
    }
}
```

Ручной запрос

    getent passwd
