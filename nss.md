# Name Service Switch

Изначально программы использовали фиксированные источники данных. В какой-то момент решили перенести этот функционал в отдельную библиотеку (GNU libc) с возможность настройки источников через файл nsswitch.conf.

Пример использования в программах

```c
#include <stdio.h>
#include <pwd.h>

int main(void)
{
        struct passwd *p;
        uid_t uid = 1000;

        if ((p = getpwuid(uid)) == 0)
                perror("getpwuid() error");
        else {
                printf("getpwuid() return struct for uid %d:\n", uid);
                printf("    pw_name   : %s\n", p->pw_name);
                printf("    pw_passwd : %s\n", p->pw_passwd);
                printf("    pw_uid    : %d\n", p->pw_uid);
                printf("    pw_gid    : %d\n", p->pw_gid);
                printf("    pw_gecos  : %s\n", p->pw_gecos);
                printf("    pw_dir    : %s\n", p->pw_dir);
                printf("    pw_shell  : %s\n", p->pw_shell);
        }
}
```

Команда для запроса данных

    getent passwd
