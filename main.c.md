```
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <windows.h>
#include "team_db.h"

int main()
{
    sqlite3* db = NULL;
    char input[100];
    char* end;
    long value;
    int menu;

    // 콘솔 한글 입출력 설정
    SetConsoleOutputCP(CP_UTF8);
    SetConsoleCP(CP_UTF8);

    // 데이터베이스 연결
    if (db_open(&db) != 0)
    {
        printf("프로그램을 종료합니다.\n");
        return 1;
    }

    // 처음 실행했을 때만 팀원 3명 등록
    insert_initial_members(db);

    while (1)
    {
        printf("\n===== 팀 연락처 관리 및 일정 관리 프로그램 =====\n");
        printf("1. 전체 정보 조회\n");
        printf("2. 이름 검색\n");
        printf("3. 전화번호 수정\n");
        printf("4. 정보 삭제\n");
        printf("5. 정보 추가\n");
        printf("6. 정보 초기화\n");
        printf("7. 팀원 통계\n");
        printf("8. 일정 추가\n");
        printf("9. 전체 일정 조회\n");
        printf("10. 일정 삭제\n");
        printf("11. 팀 공통 메모 추가\n");
        printf("12. 팀 공통 메모 조회\n");
        printf("13. 팀 공통 메모 삭제\n");
        printf("0. 종료\n");
        printf("메뉴 선택: ");

        // 메뉴 입력
        if (fgets(input, sizeof(input), stdin) == NULL)
        {
            printf("\n입력을 종료합니다.\n");
            break;
        }

        errno = 0;
        value = strtol(input, &end, 10);

        // 숫자가 아닌 입력 확인
        if (end == input || errno == ERANGE)
        {
            printf("잘못된 입력입니다. 숫자를 입력해주세요.\n");
            continue;
        }

        // 숫자 뒤에 다른 문자가 있는지 확인
        while (*end == ' ' || *end == '\t' ||
            *end == '\r' || *end == '\n')
        {
            end++;
        }

        if (*end != '\0' || value < 0 || value > 13)
        {
            printf("0부터 13까지의 메뉴 번호를 입력해주세요.\n");
            continue;
        }

        menu = (int)value;

        switch (menu)
        {
        case 1:
            show_members(db);
            break;

        case 2:
            search_member(db);
            break;

        case 3:
            update_member(db);
            break;

        case 4:
            delete_member(db);
            break;

        case 5:
            add_member(db);
            break;

        case 6:
            reset_members(db);
            break;

        case 7:
            show_statistics(db);
            break;

        case 8:
            add_schedule(db);
            break;

        case 9:
            show_schedules(db);
            break;

        case 10:
            delete_schedule(db);
            break;

        case 11:
            add_team_memo(db);
            break;

        case 12:
            show_team_memos(db);
            break;

        case 13:
            delete_team_memo(db);
            break;

        case 0:
            printf("프로그램을 종료합니다.\n");
            db_close(db);
            return 0;
        }
    }

    db_close(db);
    return 0;
}

```
