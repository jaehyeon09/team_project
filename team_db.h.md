```
#ifndef TEAM_DB_H
#define TEAM_DB_H

#include "sqlite3.h"

int db_open(sqlite3** db);
void db_close(sqlite3* db);
void insert_initial_members(sqlite3* db);

void show_members(sqlite3* db);
void search_member(sqlite3* db);
void update_member(sqlite3* db);
void delete_member(sqlite3* db);
void add_member(sqlite3* db);
void reset_members(sqlite3* db);
void show_statistics(sqlite3* db);

// 일정 관리
void add_schedule(sqlite3* db);
void show_schedules(sqlite3* db);
void delete_schedule(sqlite3* db);

// 팀 공통 메모(추가,삭제,조회) 기능
void add_team_memo(sqlite3* db);
void show_team_memos(sqlite3* db);
void delete_team_memo(sqlite3* db);

#endif
```
