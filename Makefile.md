```
CC = cl
CFLAGS = /nologo /W4 /utf-8

TARGET = team_manager.exe
OBJS = main.obj team_db.obj sqlite3.obj

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) /Fe:$(TARGET) $(OBJS)

main.obj: main.c team_db.h
	$(CC) $(CFLAGS) /c main.c

team_db.obj: team_db.c team_db.h
	$(CC) $(CFLAGS) /c team_db.c

sqlite3.obj: sqlite3.c sqlite3.h
	$(CC) $(CFLAGS) /c sqlite3.c

clean:
	del /Q *.obj $(TARGET)


```
