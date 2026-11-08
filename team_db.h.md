```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <errno.h>
#include <limits.h>
#include "team_db.h"

// 문자열 입력 검사
static int input_line(const char* message, char* buffer, int size)
{
    size_t len;
    int ch;

    printf("%s", message);

    if (fgets(buffer, size, stdin) == NULL)
    {
        printf("입력 오류가 발생했습니다.\n");
        return 0;
    }

    len = strlen(buffer);

    if (len > 0 && buffer[len - 1] == '\n')
    {
        buffer[len - 1] = '\0';
    }
    else if (!feof(stdin))
    {
        while ((ch = getchar()) != '\n' && ch != EOF);
        printf("입력 내용이 너무 깁니다.\n");
        return 0;
    }

    if (buffer[0] == '\0')
    {
        printf("빈 값은 입력할 수 없습니다.\n");
        return 0;
    }

    return 1;
}

// 숫자 입력 검사
static int input_int(const char* message, int* number)
{
    char input[100];
    char* end;
    long value;

    if (!input_line(message, input, sizeof(input)))
        return 0;

    errno = 0;
    value = strtol(input, &end, 10);

    if (end == input || errno == ERANGE ||
        value < INT_MIN || value > INT_MAX)
    {
        printf("올바른 숫자를 입력해주세요.\n");
        return 0;
    }

    while (isspace((unsigned char)*end))
        end++;

    if (*end != '\0')
    {
        printf("숫자만 입력해주세요.\n");
        return 0;
    }

    *number = (int)value;
    return 1;
}

// SQL 준비 및 오류 확인
static int prepare_sql(sqlite3* db, const char* sql,
    sqlite3_stmt** stmt)
{
    if (sqlite3_prepare_v2(db, sql, -1, stmt, NULL) != SQLITE_OK)
    {
        printf("DB 오류: %s\n", sqlite3_errmsg(db));
        return 0;
    }

    return 1;
}

// 문자열 바인딩
static int bind_text_value(sqlite3* db, sqlite3_stmt* stmt,
    int index, const char* value)
{
    if (sqlite3_bind_text(stmt, index, value, -1,
        SQLITE_TRANSIENT) != SQLITE_OK)
    {
        printf("DB 입력 오류: %s\n", sqlite3_errmsg(db));
        return 0;
    }

    return 1;
}

// DB 연결 및 테이블 생성
int db_open(sqlite3** db)
{
    char* err = NULL;

    // DB 연결
    if (sqlite3_open("team.db", db) != SQLITE_OK)
    {
        printf("DB 연결 실패: %s\n",
            *db != NULL ? sqlite3_errmsg(*db) : "알 수 없는 오류");

        if (*db != NULL)
            sqlite3_close(*db);

        *db = NULL;
        return 1;
    }

    printf("DB 연결 성공!\n");

    // 테이블 생성 및 외래키 기능 활성화
    const char* sql =
        "PRAGMA foreign_keys = ON;"

        // 기존 팀원 테이블
        "CREATE TABLE IF NOT EXISTS team("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "name TEXT,"
        "phone TEXT,"
        "memo TEXT);"

        // 팀원별 일정 테이블
        "CREATE TABLE IF NOT EXISTS schedule("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "team_id INTEGER NOT NULL,"
        "title TEXT NOT NULL,"
        "schedule_date TEXT NOT NULL,"
        "description TEXT,"
        "FOREIGN KEY(team_id) REFERENCES team(id) "
        "ON DELETE CASCADE);"

        // 팀 공통 메모 테이블
        "CREATE TABLE IF NOT EXISTS team_memo("
        "id INTEGER PRIMARY KEY AUTOINCREMENT,"
        "content TEXT NOT NULL,"
        "created_at TEXT DEFAULT CURRENT_TIMESTAMP);";

    // SQL 실행
    if (sqlite3_exec(*db, sql, NULL, NULL, &err) != SQLITE_OK)
    {
        printf("테이블 생성 실패: %s\n",
            err != NULL ? err : sqlite3_errmsg(*db));

        sqlite3_free(err);
        sqlite3_close(*db);
        *db = NULL;

        return 1;
    }

    printf("데이터베이스 테이블 확인 완료!\n");

    return 0;
}

// DB 종료
void db_close(sqlite3* db)
{
    if (db != NULL)
    {
        if (sqlite3_close(db) != SQLITE_OK)
        {
            printf("DB 종료 중 오류가 발생했습니다.\n");
        }
    }
}

// 초기 팀원 등록
void insert_initial_members(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int count = 0;
    int i;
    int result;

    if (!prepare_sql(db, "SELECT COUNT(*) FROM team;", &stmt))
        return;

    result = sqlite3_step(stmt);

    if (result == SQLITE_ROW)
        count = sqlite3_column_int(stmt, 0);
    else
    {
        printf("초기 데이터 확인 실패: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    sqlite3_finalize(stmt);

    if (count > 0)
        return;

    printf("\n===== 초기 팀원 등록 =====\n");

    for (i = 0; i < 3; i++)
    {
        char name[50];
        char phone[50];
        char memo[100];

        char sql[] =
            "INSERT INTO team(name, phone, memo) VALUES(?, ?, ?);";

        printf("\n%d번째 팀원\n", i + 1);

        if (!input_line("이름: ", name, sizeof(name)))
            return;

        if (!input_line("전화번호: ", phone, sizeof(phone)))
            return;

        if (!input_line("메모: ", memo, sizeof(memo)))
            return;

        if (!prepare_sql(db, sql, &stmt))
            return;

        if (!bind_text_value(db, stmt, 1, name) ||
            !bind_text_value(db, stmt, 2, phone) ||
            !bind_text_value(db, stmt, 3, memo))
        {
            sqlite3_finalize(stmt);
            return;
        }

        result = sqlite3_step(stmt);

        if (result == SQLITE_DONE)
            printf("팀원 등록 완료!\n");
        else
            printf("팀원 등록 실패: %s\n", sqlite3_errmsg(db));

        sqlite3_finalize(stmt);

        if (result != SQLITE_DONE)
            return;
    }
}

// 전체 정보 조회
void show_members(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int found = 0;
    int result;

    if (!prepare_sql(db, "SELECT * FROM team ORDER BY id ASC;", &stmt))
        return;

    printf("\n===== 전체 팀원 정보 =====\n");

    while ((result = sqlite3_step(stmt)) == SQLITE_ROW)
    {
        found = 1;

        printf("\n번호: %d\n", sqlite3_column_int(stmt, 0));
        printf("이름: %s\n",
            (const char*)sqlite3_column_text(stmt, 1));
        printf("전화번호: %s\n",
            (const char*)sqlite3_column_text(stmt, 2));
        printf("메모: %s\n",
            (const char*)sqlite3_column_text(stmt, 3));
    }

    if (result != SQLITE_DONE)
        printf("조회 실패: %s\n", sqlite3_errmsg(db));
    else if (!found)
        printf("등록된 정보가 없습니다.\n");

    sqlite3_finalize(stmt);
}

// 이름 검색
void search_member(sqlite3* db)
{
    sqlite3_stmt* stmt;
    char name[50];
    int found = 0;
    int result;

    if (!input_line("검색할 이름: ", name, sizeof(name)))
        return;

    if (!prepare_sql(db, "SELECT * FROM team WHERE name=?;", &stmt))
        return;

    if (!bind_text_value(db, stmt, 1, name))
    {
        sqlite3_finalize(stmt);
        return;
    }

    while ((result = sqlite3_step(stmt)) == SQLITE_ROW)
    {
        found = 1;

        printf("\n번호: %d\n", sqlite3_column_int(stmt, 0));
        printf("이름: %s\n",
            (const char*)sqlite3_column_text(stmt, 1));
        printf("전화번호: %s\n",
            (const char*)sqlite3_column_text(stmt, 2));
        printf("메모: %s\n",
            (const char*)sqlite3_column_text(stmt, 3));
    }

    if (result != SQLITE_DONE)
        printf("검색 실패: %s\n", sqlite3_errmsg(db));
    else if (!found)
        printf("검색 결과가 없습니다.\n");

    sqlite3_finalize(stmt);
}

// 전화번호 수정
void update_member(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int id;
    int result;
    char phone[50];

    if (!input_int("수정할 정보의 번호: ", &id))
        return;

    if (id <= 0)
    {
        printf("번호는 1 이상의 숫자여야 합니다.\n");
        return;
    }

    if (!input_line("새로운 전화번호: ", phone, sizeof(phone)))
        return;

    if (!prepare_sql(db, "UPDATE team SET phone=? WHERE id=?;", &stmt))
        return;

    if (!bind_text_value(db, stmt, 1, phone) ||
        sqlite3_bind_int(stmt, 2, id) != SQLITE_OK)
    {
        printf("DB 입력 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    result = sqlite3_step(stmt);

    if (result == SQLITE_DONE)
    {
        if (sqlite3_changes(db) > 0)
            printf("정보 수정 완료!\n");
        else
            printf("해당 번호의 정보가 없습니다.\n");
    }
    else
        printf("정보 수정 실패: %s\n", sqlite3_errmsg(db));

    sqlite3_finalize(stmt);
}

// SQL 실행 보조 함수
static int execute_sql_checked(sqlite3* db, const char* sql,
    const char* error_message)
{
    char* err = NULL;
    if (sqlite3_exec(db, sql, NULL, NULL, &err) != SQLITE_OK)
    {
        printf("%s: %s\n", error_message,
            err != NULL ? err : sqlite3_errmsg(db));
        sqlite3_free(err);
        return 0;
    }
    sqlite3_free(err);
    return 1;
}

// 일정 번호를 1번부터 다시 정렬
static int compact_schedule_ids(sqlite3* db)
{
    const char* sql =
        "UPDATE schedule SET id = -id;"
        "CREATE TEMP TABLE IF NOT EXISTS schedule_id_map("
        "old_id INTEGER PRIMARY KEY, new_id INTEGER);"
        "DELETE FROM schedule_id_map;"
        "INSERT INTO schedule_id_map(old_id, new_id) "
        "SELECT s.id, (SELECT COUNT(*) FROM schedule AS s2 "
        "WHERE s2.id >= s.id) FROM schedule AS s;"
        "UPDATE schedule SET id = (SELECT new_id FROM schedule_id_map "
        "WHERE old_id = schedule.id);"
        "DROP TABLE schedule_id_map;"
        "DELETE FROM sqlite_sequence WHERE name='schedule';"
        "INSERT INTO sqlite_sequence(name, seq) "
        "SELECT 'schedule', COALESCE(MAX(id), 0) FROM schedule;";
    return execute_sql_checked(db, sql, "일정 번호 재정렬 실패");
}

// 공통 메모 번호를 1번부터 다시 정렬
static int compact_team_memo_ids(sqlite3* db)
{
    const char* sql =
        "UPDATE team_memo SET id = -id;"
        "CREATE TEMP TABLE IF NOT EXISTS memo_id_map("
        "old_id INTEGER PRIMARY KEY, new_id INTEGER);"
        "DELETE FROM memo_id_map;"
        "INSERT INTO memo_id_map(old_id, new_id) "
        "SELECT m.id, (SELECT COUNT(*) FROM team_memo AS m2 "
        "WHERE m2.id >= m.id) FROM team_memo AS m;"
        "UPDATE team_memo SET id = (SELECT new_id FROM memo_id_map "
        "WHERE old_id = team_memo.id);"
        "DROP TABLE memo_id_map;"
        "DELETE FROM sqlite_sequence WHERE name='team_memo';"
        "INSERT INTO sqlite_sequence(name, seq) "
        "SELECT 'team_memo', COALESCE(MAX(id), 0) FROM team_memo;";
    return execute_sql_checked(db, sql, "공통 메모 번호 재정렬 실패");
}

// 팀원 번호와 해당 팀원의 일정 연결을 함께 재정렬
static int compact_team_ids_and_schedule_links(sqlite3* db)
{
    const char* sql =
        "UPDATE team SET id = -id;"
        "UPDATE schedule SET team_id = -team_id;"
        "CREATE TEMP TABLE IF NOT EXISTS team_id_map("
        "old_id INTEGER PRIMARY KEY, new_id INTEGER);"
        "DELETE FROM team_id_map;"
        "INSERT INTO team_id_map(old_id, new_id) "
        "SELECT t.id, (SELECT COUNT(*) FROM team AS t2 "
        "WHERE t2.id >= t.id) FROM team AS t;"
        "UPDATE schedule SET team_id = (SELECT new_id FROM team_id_map "
        "WHERE old_id = schedule.team_id);"
        "UPDATE team SET id = (SELECT new_id FROM team_id_map "
        "WHERE old_id = team.id);"
        "DROP TABLE team_id_map;"
        "DELETE FROM sqlite_sequence WHERE name='team';"
        "INSERT INTO sqlite_sequence(name, seq) "
        "SELECT 'team', COALESCE(MAX(id), 0) FROM team;";
    return execute_sql_checked(db, sql, "팀원 번호 및 일정 연결 재정렬 실패");
}

// 정보 삭제 및 팀원 번호 재정렬
void delete_member(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int id;
    int result;

    if (!input_int("삭제할 정보의 번호: ", &id))
        return;
    if (id <= 0)
    {
        printf("번호는 1 이상의 숫자여야 합니다.\n");
        return;
    }

    if (!execute_sql_checked(db,
        "BEGIN IMMEDIATE; PRAGMA defer_foreign_keys = ON;",
        "삭제 작업 시작 실패"))
        return;

    if (!prepare_sql(db, "DELETE FROM team WHERE id=?;", &stmt))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }

    if (sqlite3_bind_int(stmt, 1, id) != SQLITE_OK)
    {
        printf("DB 입력 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }

    result = sqlite3_step(stmt);
    if (result != SQLITE_DONE)
    {
        printf("정보 삭제 실패: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }

    if (sqlite3_changes(db) == 0)
    {
        printf("해당 번호의 정보가 없습니다.\n");
        sqlite3_finalize(stmt);
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }
    sqlite3_finalize(stmt);

    if (!compact_team_ids_and_schedule_links(db) ||
        !compact_schedule_ids(db))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        printf("오류로 인해 삭제 작업을 취소했습니다.\\n");
        return;
    }

    if (!execute_sql_checked(db, "COMMIT;", "삭제 작업 완료 실패"))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        printf("오류로 인해 삭제 작업을 취소했습니다.\n");
        return;
    }

    printf("정보 삭제 및 팀원 번호 재정렬이 완료되었습니다.\n");
    printf("담당 일정 연결도 함께 변경되었습니다.\n");
}

// 정보 추가
void add_member(sqlite3* db)
{
    sqlite3_stmt* stmt;
    char name[50];
    char phone[50];
    char memo[100];
    int result;

    if (!input_line("이름: ", name, sizeof(name)))
        return;

    if (!input_line("전화번호: ", phone, sizeof(phone)))
        return;

    if (!input_line("메모: ", memo, sizeof(memo)))
        return;

    if (!prepare_sql(db,
        "INSERT INTO team(name, phone, memo) VALUES(?, ?, ?);",
        &stmt))
        return;

    if (!bind_text_value(db, stmt, 1, name) ||
        !bind_text_value(db, stmt, 2, phone) ||
        !bind_text_value(db, stmt, 3, memo))
    {
        sqlite3_finalize(stmt);
        return;
    }

    result = sqlite3_step(stmt);

    if (result == SQLITE_DONE)
        printf("정보 추가 완료!\n");
    else
        printf("정보 추가 실패: %s\n", sqlite3_errmsg(db));

    sqlite3_finalize(stmt);
}

// 정보 초기화
void reset_members(sqlite3* db)
{
    int check;
    if (!input_int("초기화하려면 1, 취소하려면 0 입력: ", &check))
        return;

    if (check == 1)
    {
        if (!execute_sql_checked(db, "BEGIN IMMEDIATE;",
            "초기화 작업 시작 실패"))
            return;

        if (!execute_sql_checked(db,
            "DELETE FROM team;"
            "DELETE FROM sqlite_sequence WHERE name='team';"
            "INSERT INTO sqlite_sequence(name, seq) VALUES('team', 0);"
            "DELETE FROM sqlite_sequence WHERE name='schedule';"
            "INSERT INTO sqlite_sequence(name, seq) VALUES('schedule', 0);",
            "정보 초기화 실패"))
        {
            sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
            printf("오류로 인해 초기화를 취소했습니다.\n");
            return;
        }

        if (!execute_sql_checked(db, "COMMIT;", "초기화 완료 실패"))
        {
            sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
            printf("오류로 인해 초기화를 취소했습니다.\n");
            return;
        }
        printf("모든 팀원 정보와 연결된 일정, 번호가 초기화되었습니다.\n");
    }
    else if (check == 0)
        printf("초기화를 취소했습니다.\n");
    else
        printf("1 또는 0만 입력해주세요.\n");
}

// 전체 통계 조회
void show_statistics(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int result;

    int member_count = 0;
    int schedule_count = 0;
    int memo_count = 0;

    printf("\n===== 프로그램 전체 통계 =====\n");

    // 팀원, 일정, 공통 메모 개수 조회
    if (!prepare_sql(db,
        "SELECT "
        "(SELECT COUNT(*) FROM team), "
        "(SELECT COUNT(*) FROM schedule), "
        "(SELECT COUNT(*) FROM team_memo);",
        &stmt))
    {
        return;
    }

    result = sqlite3_step(stmt);

    if (result == SQLITE_ROW)
    {
        member_count = sqlite3_column_int(stmt, 0);
        schedule_count = sqlite3_column_int(stmt, 1);
        memo_count = sqlite3_column_int(stmt, 2);

        printf("\n------------------------------\n");
        printf("전체 팀원 수: %d명\n", member_count);
        printf("전체 일정 수: %d개\n", schedule_count);
        printf("공통 메모 수: %d개\n", memo_count);
        printf("------------------------------\n");

        printf("\n===== 통계 조회 완료 =====\n");
    }
    else
    {
        printf("통계 조회 실패: %s\n", sqlite3_errmsg(db));
    }

    sqlite3_finalize(stmt);
}

// 일정 추가
void add_schedule(sqlite3* db)
{
    sqlite3_stmt* stmt;

    int team_id;
    int result;

    char title[100];
    char schedule_date[20];
    char description[200];

    char team_name[100];

    // 팀원 번호 입력
    printf("\n===== 일정 추가 =====\n");

    if (!input_int("일정을 등록할 팀원 번호: ", &team_id))
        return;

    if (team_id <= 0)
    {
        printf("팀원 번호는 1 이상의 숫자여야 합니다.\n");
        return;
    }

    // 해당 팀원이 존재하는지 확인
    if (!prepare_sql(db,
        "SELECT name FROM team WHERE id=?;",
        &stmt))
        return;

    if (sqlite3_bind_int(stmt, 1, team_id) != SQLITE_OK)
    {
        printf("DB 입력 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    result = sqlite3_step(stmt);

    if (result == SQLITE_ROW)
    {
        const unsigned char* name = sqlite3_column_text(stmt, 0);

        snprintf(team_name, sizeof(team_name), "%s",
            name != NULL ? (const char*)name : "");
    }
    else if (result == SQLITE_DONE)
    {
        printf("해당 번호의 팀원이 존재하지 않습니다.\n");
        sqlite3_finalize(stmt);
        return;
    }
    else
    {
        printf("팀원 확인 실패: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    sqlite3_finalize(stmt);

    printf("\n등록 대상 팀원: %s\n", team_name);

    // 일정 정보 입력
    if (!input_line("일정 제목: ", title, sizeof(title)))
        return;

    if (!input_line("일정 날짜(YYYY-MM-DD): ",
        schedule_date, sizeof(schedule_date)))
        return;

    // 날짜 형식 검사
    if (strlen(schedule_date) != 10 ||
        schedule_date[4] != '-' ||
        schedule_date[7] != '-')
    {
        printf("날짜 형식이 올바르지 않습니다.\n");
        printf("예시: 2026-11-10\n");
        return;
    }

    for (int i = 0; i < 10; i++)
    {
        if (i == 4 || i == 7)
            continue;

        if (!isdigit((unsigned char)schedule_date[i]))
        {
            printf("날짜에는 숫자와 '-'만 입력할 수 있습니다.\n");
            return;
        }
    }

    if (!input_line("일정 상세 내용: ",
        description, sizeof(description)))
        return;

    // 일정 데이터 저장
    if (!prepare_sql(db,
        "INSERT INTO schedule(team_id, title, schedule_date, description) "
        "VALUES(?, ?, ?, ?);",
        &stmt))
        return;

    if (sqlite3_bind_int(stmt, 1, team_id) != SQLITE_OK ||
        !bind_text_value(db, stmt, 2, title) ||
        !bind_text_value(db, stmt, 3, schedule_date) ||
        !bind_text_value(db, stmt, 4, description))
    {
        printf("일정 입력 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    result = sqlite3_step(stmt);

    if (result == SQLITE_DONE)
        printf("\n일정 등록 완료!\n");
    else
        printf("일정 등록 실패: %s\n", sqlite3_errmsg(db));

    sqlite3_finalize(stmt);
}
// 전체 일정 조회
void show_schedules(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int result;
    int count = 0;

    printf("\n===== 전체 일정 조회 =====\n");

    // 팀원 정보와 일정 정보를 함께 조회
    const char* sql =
        "SELECT schedule.id, team.name, "
        "schedule.title, schedule.schedule_date, "
        "schedule.description "
        "FROM schedule "
        "JOIN team ON schedule.team_id = team.id "
        "ORDER BY schedule.schedule_date ASC;";

    if (!prepare_sql(db, sql, &stmt))
        return;

    while ((result = sqlite3_step(stmt)) == SQLITE_ROW)
    {
        count++;

        int id = sqlite3_column_int(stmt, 0);

        const unsigned char* name =
            sqlite3_column_text(stmt, 1);

        const unsigned char* title =
            sqlite3_column_text(stmt, 2);

        const unsigned char* date =
            sqlite3_column_text(stmt, 3);

        const unsigned char* description =
            sqlite3_column_text(stmt, 4);

        printf("\n[%d번째 일정]\n", count);
        printf("일정 번호: %d\n", id);
        printf("담당 팀원: %s\n",
            name ? (const char*)name : "");

        printf("일정 제목: %s\n",
            title ? (const char*)title : "");

        printf("일정 날짜: %s\n",
            date ? (const char*)date : "");

        printf("상세 내용: %s\n",
            description ? (const char*)description : "");

        printf("--------------------------\n");
    }

    if (result != SQLITE_DONE)
    {
        printf("일정 조회 실패: %s\n", sqlite3_errmsg(db));
    }
    else if (count == 0)
    {
        printf("등록된 일정이 없습니다.\n");
    }
    else
    {
        printf("\n총 %d개의 일정이 조회되었습니다.\n", count);
    }

    sqlite3_finalize(stmt);
}
// 일정 삭제 및 일정 번호 재정렬
void delete_schedule(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int schedule_id;
    int result;

    printf("\n===== 일정 삭제 =====\n");
    if (!input_int("삭제할 일정 번호: ", &schedule_id))
        return;
    if (schedule_id <= 0)
    {
        printf("일정 번호는 1 이상의 숫자여야 합니다.\n");
        return;
    }

    if (!prepare_sql(db, "SELECT title FROM schedule WHERE id=?;", &stmt))
        return;
    if (sqlite3_bind_int(stmt, 1, schedule_id) != SQLITE_OK)
    {
        printf("일정 확인 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    result = sqlite3_step(stmt);
    if (result == SQLITE_ROW)
    {
        const unsigned char* title = sqlite3_column_text(stmt, 0);
        printf("삭제 대상 일정: %s\n",
            title ? (const char*)title : "");
    }
    else if (result == SQLITE_DONE)
    {
        printf("해당 번호의 일정이 존재하지 않습니다.\n");
        sqlite3_finalize(stmt);
        return;
    }
    else
    {
        printf("일정 확인 실패: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }
    sqlite3_finalize(stmt);

    if (!execute_sql_checked(db, "BEGIN IMMEDIATE;",
        "일정 삭제 작업 시작 실패"))
        return;

    if (!prepare_sql(db, "DELETE FROM schedule WHERE id=?;", &stmt))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }
    if (sqlite3_bind_int(stmt, 1, schedule_id) != SQLITE_OK)
    {
        printf("일정 삭제 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }

    result = sqlite3_step(stmt);
    sqlite3_finalize(stmt);
    if (result != SQLITE_DONE)
    {
        printf("일정 삭제 실패: %s\n", sqlite3_errmsg(db));
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }

    if (!compact_schedule_ids(db))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        printf("오류로 인해 일정 삭제 작업을 취소했습니다.\n");
        return;
    }
    if (!execute_sql_checked(db, "COMMIT;", "일정 삭제 완료 실패"))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        printf("오류로 인해 일정 삭제 작업을 취소했습니다.\n");
        return;
    }
    printf("일정 삭제 및 번호 재정렬이 완료되었습니다.\n");
}

// 팀 공통 메모 추가
void add_team_memo(sqlite3* db)
{
    sqlite3_stmt* stmt;
    char content[300];
    int result;

    printf("\n===== 팀 공통 메모 추가 =====\n");

    if (!input_line("메모 내용: ", content, sizeof(content)))
        return;

    if (!prepare_sql(db,
        "INSERT INTO team_memo(content) VALUES(?);",
        &stmt))
    {
        return;
    }

    if (!bind_text_value(db, stmt, 1, content))
    {
        printf("메모 입력 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    result = sqlite3_step(stmt);

    if (result == SQLITE_DONE)
        printf("팀 공통 메모가 등록되었습니다.\n");
    else
        printf("메모 등록 실패: %s\n", sqlite3_errmsg(db));

    sqlite3_finalize(stmt);
}
// 팀 공통 메모 전체 조회
void show_team_memos(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int result;
    int count = 0;

    printf("\n===== 팀 공통 메모 조회 =====\n");

    if (!prepare_sql(db,
        "SELECT id, content, created_at "
        "FROM team_memo ORDER BY id DESC;",
        &stmt))
    {
        return;
    }

    while ((result = sqlite3_step(stmt)) == SQLITE_ROW)
    {
        int id = sqlite3_column_int(stmt, 0);

        const unsigned char* content =
            sqlite3_column_text(stmt, 1);

        const unsigned char* created_at =
            sqlite3_column_text(stmt, 2);

        printf("\n------------------------------\n");
        printf("메모 번호: %d\n", id);
        printf("메모 내용: %s\n",
            content ? (const char*)content : "");
        printf("등록 시간: %s\n",
            created_at ? (const char*)created_at : "");

        count++;
    }

    if (result != SQLITE_DONE)
    {
        printf("메모 조회 실패: %s\n", sqlite3_errmsg(db));
    }
    else if (count == 0)
    {
        printf("등록된 공통 메모가 없습니다.\n");
    }
    else
    {
        printf("\n총 %d개의 메모가 조회되었습니다.\n", count);
    }

    sqlite3_finalize(stmt);
}
// 팀 공통 메모 삭제 및 번호 재정렬
void delete_team_memo(sqlite3* db)
{
    sqlite3_stmt* stmt;
    int memo_id;
    int result;

    printf("\n===== 팀 공통 메모 삭제 =====\n");
    if (!input_int("삭제할 메모 번호: ", &memo_id))
        return;
    if (memo_id <= 0)
    {
        printf("메모 번호는 1 이상의 숫자여야 합니다.\n");
        return;
    }

    if (!prepare_sql(db, "SELECT content FROM team_memo WHERE id=?;", &stmt))
        return;
    if (sqlite3_bind_int(stmt, 1, memo_id) != SQLITE_OK)
    {
        printf("메모 확인 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }

    result = sqlite3_step(stmt);
    if (result == SQLITE_ROW)
    {
        const unsigned char* content = sqlite3_column_text(stmt, 0);
        printf("삭제 대상 메모: %s\n",
            content ? (const char*)content : "");
    }
    else if (result == SQLITE_DONE)
    {
        printf("해당 번호의 메모가 존재하지 않습니다.\n");
        sqlite3_finalize(stmt);
        return;
    }
    else
    {
        printf("메모 확인 실패: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        return;
    }
    sqlite3_finalize(stmt);

    if (!execute_sql_checked(db, "BEGIN IMMEDIATE;",
        "메모 삭제 작업 시작 실패"))
        return;

    if (!prepare_sql(db, "DELETE FROM team_memo WHERE id=?;", &stmt))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }
    if (sqlite3_bind_int(stmt, 1, memo_id) != SQLITE_OK)
    {
        printf("메모 삭제 오류: %s\n", sqlite3_errmsg(db));
        sqlite3_finalize(stmt);
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }

    result = sqlite3_step(stmt);
    sqlite3_finalize(stmt);
    if (result != SQLITE_DONE)
    {
        printf("메모 삭제 실패: %s\n", sqlite3_errmsg(db));
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        return;
    }

    if (!compact_team_memo_ids(db))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        printf("오류로 인해 메모 삭제 작업을 취소했습니다.\n");
        return;
    }
    if (!execute_sql_checked(db, "COMMIT;", "메모 삭제 완료 실패"))
    {
        sqlite3_exec(db, "ROLLBACK;", NULL, NULL, NULL);
        printf("오류로 인해 메모 삭제 작업을 취소했습니다.\n");
        return;
    }
    printf("팀 공통 메모 삭제 및 번호 재정렬이 완료되었습니다.\n");
}

```
