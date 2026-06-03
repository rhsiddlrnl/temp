#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <windows.h>
#include <conio.h>
#include <time.h>
#include <stdlib.h>
#include <string.h>

#define TOTAL_STAGE_COUNT 4 

typedef struct {
    int width;
    int height;
    int tile[50][80];
    int timeLimit;
    int startRow;
    int startCol;
    char levelName[40];
} Map;

Map map[TOTAL_STAGE_COUNT];

// 함수 원형 선언
void loadAllMaps();
void initRandomMaze(int stage, int w, int h, int limit, const char* name);
void makeMazeDFS(int r, int c, int stage);
void showMap(int stage);
void move(char direction, int* row, int* column, int stage, int* hasKey);
int playGame(int stage);
void showStartScreen();
void showEndingScreen();
void showClearScreen(int stage);
void exportMapToFile(int stage);
void importMapFromFile(int stage, const char* filename, int limit, const char* name);

int hasStarted = 0;

int main() {
    showStartScreen();

    // 랜덤 맵 생성을 위한 시드 설정
    srand(time(NULL));

    // 전체 맵 세팅 (커스텀 로드 + 랜덤 생성)
    loadAllMaps();

    // 스테이지 순차 실행
    for (int i = 0; i < TOTAL_STAGE_COUNT; i++) {
        if (map[i].width == 0 || map[i].height == 0) {
            break;
        }

        system("cls");
        printf("\033[5;5f=== [%s] 진입! ===\n", map[i].levelName);
        Sleep(1500);
        system("cls");

        int result = playGame(i);

        if (result == 1) {
            showClearScreen(i + 1);
        }
        else {
            system("cls");
            printf("\n\n  탈출 실패... 미로에 갇혔습니다.\n");
            return 0;
        }
    }

    system("cls");
    showEndingScreen();
    Sleep(3000);

    return 0;
}

void loadAllMaps() {

    //importMapFromFile(0, "map_export_stage_1.txt", 120, "Stage 1");

    initRandomMaze(0, 79, 43, 150, "Stage 1");
}

void initRandomMaze(int stage, int w, int h, int limit, const char* name) {
    map[stage].width = w;   //맵 너비
    map[stage].height = h;  //맵 높이
    map[stage].timeLimit = limit;   //시간제한
    map[stage].startRow = 1;    //플레이어 시작 x좌표
    map[stage].startCol = 1;    //플레이어 시작 y좌표
    strcpy(map[stage].levelName, name); //스테이지 이름

    for (int i = 0; i < h; i++) {
        for (int j = 0; j < w; j++) {
            map[stage].tile[i][j] = 1;  //너비 * 높이 만큼 1로 초기화
        }
    }

    makeMazeDFS(1, 1, stage);   //미로 생성 함수

    int exitRow = h - 2;    //출구 y좌표
    int exitCol = w - 2;    //출구 x좌표
    map[stage].tile[exitRow][exitCol] = 2; // 도착 지점(탈출구)

    int keyRow, keyCol; //열쇠 생성 x좌표, y좌표
    while (1) {
        keyRow = (rand() % (h / 2)) * 2 + 1;
        keyCol = (rand() % (w / 2)) * 2 + 1;

        if (map[stage].tile[keyRow][keyCol] == 0) {
            map[stage].tile[keyRow][keyCol] = 4; // 4: 열쇠
            break;
        }
    }


    //탈출구 주변 문으로 막기(열쇠가 있어야 클리어 가능하도록)
    // 위
    if (map[stage].tile[exitRow - 1][exitCol] == 0) {
        map[stage].tile[exitRow - 1][exitCol] = 5; // 5: 닫힌 문
    }
    // 왼쪽
    if (map[stage].tile[exitRow][exitCol - 1] == 0) {
        map[stage].tile[exitRow][exitCol - 1] = 5;
    }
    // 아래
    if (map[stage].tile[exitRow + 1][exitCol] == 0) {
        map[stage].tile[exitRow + 1][exitCol] = 5;
    }
    // 오른쪽
    if (map[stage].tile[exitRow][exitCol + 1] == 0) {
        map[stage].tile[exitRow][exitCol + 1] = 5;
    }
    // ----------------------------------------------------


    //시간 추가 아이템 배치 로직
    int targetItemCount = (w * h) / 150;
    if (targetItemCount < 3) targetItemCount = 3;

    int itemsPlaced = 0;
    while (itemsPlaced < targetItemCount) {
        int r = (rand() % (h / 2)) * 2 + 1;
        int c = (rand() % (w / 2)) * 2 + 1;

        if ((r != 1 || c != 1) && (r != exitRow || c != exitCol)) {
            if (map[stage].tile[r][c] == 0) {
                map[stage].tile[r][c] = 3;
                itemsPlaced++;
            }
        }
    }

    //랜덤 생성된 맵 txt 파일로 출력(직접 수정용)
    exportMapToFile(stage);
}

//미로 랜덤 생성 함수
void makeMazeDFS(int r, int c, int stage) {
    map[stage].tile[r][c] = 0;

    int dr[4] = { -2, 2, 0, 0 };
    int dc[4] = { 0, 0, -2, 2 };

    int order[4] = { 0, 1, 2, 3 };
    for (int i = 0; i < 4; i++) {
        int dest = rand() % 4;
        int temp = order[i];
        order[i] = order[dest];
        order[dest] = temp;
    }

    for (int i = 0; i < 4; i++) {
        int dir = order[i];
        int nextR = r + dr[dir];
        int nextC = c + dc[dir];

        if (nextR > 0 && nextR < map[stage].height - 1 && nextC > 0 && nextC < map[stage].width - 1) {
            if (map[stage].tile[nextR][nextC] == 1) {
                map[stage].tile[r + dr[dir] / 2][c + dc[dir] / 2] = 0;
                makeMazeDFS(nextR, nextC, stage);
            }
        }
    }
}

//맵 출력 함수(수업자료 변형) 열쇠, 문, 아이템 추가
void showMap(int stage) {
    char symbol[10][1024] = {
        "  ",
        "\033[41m  \033[0m",
        "\033[34m문\033[0m",
        "\033[43m  \033[0m",
        "\033[33m열\033[0m",
        "\033[45m문\033[0m",
        "\033[42m  \033[0m"
    };
    printf("\033[2;1f");
    for (int i = 0; i < map[stage].height; i++) {
        for (int j = 0; j < map[stage].width; j++) {
            printf("%s", symbol[map[stage].tile[i][j]]);
        }
        printf("\n");
    }
}

//플레이어 이동함수(수업자료 변형) + 열쇠 보유시에만 문위로 이동 가능하게 수정
void move(char direction, int* row, int* column, int stage, int* hasKey) {
    if (((direction == 'w') || (direction == 'W'))
        && (*row > 0)
        && (map[stage].tile[*row - 1][*column] != 1))
    {
        if (map[stage].tile[*row - 1][*column] == 5) {
            if (*hasKey) {
                map[stage].tile[*row - 1][*column] = 0;
                *hasKey = 0;
            }
            else {
                return;
            }
        }
        (*row)--;
    }
    else if (((direction == 'a') || (direction == 'A'))
        && (*column > 0)
        && (map[stage].tile[*row][*column - 1] != 1))
    {
        if (map[stage].tile[*row][*column - 1] == 5) {
            if (*hasKey) {
                map[stage].tile[*row][*column - 1] = 0;
                *hasKey = 0;
            }
            else {
                return;
            }
        }
        (*column)--;
    }
    else if (((direction == 's') || (direction == 'S'))
        && (*row < map[stage].height - 1)
        && (map[stage].tile[*row + 1][*column] != 1))
    {
        if (map[stage].tile[*row + 1][*column] == 5) {
            if (*hasKey) {
                map[stage].tile[*row + 1][*column] = 0;
                *hasKey = 0;
            }
            else {
                return;
            }
        }
        (*row)++;
    }
    else if (((direction == 'd') || (direction == 'D'))
        && (*column < map[stage].width - 1)
        && (map[stage].tile[*row][*column + 1] != 1))
    {
        if (map[stage].tile[*row][*column + 1] == 5) {
            if (*hasKey) {
                map[stage].tile[*row][*column + 1] = 0;
                *hasKey = 0;
            }
            else {
                return;
            }
        }
        (*column)++;
    }
}

// 게임 플레이 메인 루프
int playGame(int stage) {
    int row = map[stage].startRow;
    int column = map[stage].startCol;
    time_t startTime = time(NULL);

    char eventMessage[100] = "";
    int hasKey = 0;

    showMap(stage);
    printf("\033[%d;%df옷", row + 2, column * 2 + 1);

    while (1) {
        time_t currentTime = time(NULL);
        //remaining이라는 int형 변수에 남은 시간 계속 대입
        int remaining = map[stage].timeLimit - (int)difftime(currentTime, startTime);

        //남은 시간, 열쇠 보유 여부, 조작법 출력
        printf("\033[1;1f\033[2K[%s] 남은 시간: %d초 | 열쇠: %s | (조작: WASD)    %s ",
            map[stage].levelName, remaining, (hasKey ? "O" : "X"), eventMessage);

        //남은 시간이 0보다 작거나 같아지면 시간초과로 게임 오버
        if (remaining <= 0) {
            printf("\033[%d;1f\033[2KGAME OVER! 시간 초과!\n", map[stage].height + 3);
            return 0;
        }

        //키보드 입력이 있을 때만 실행
        if (_kbhit()) {
            //getch()를 통해 enter키를 누르지 않아도 wasd를 누르자마자 변수에 대입함.
            char direction = _getch();
            printf("\033[%d;%df  ", row + 2, column * 2 + 1);

            //입력 받은 wasd에 따라 이동시키는 이동함수
            move(direction, &row, &column, stage, &hasKey);

            //입력 후 도달한 타일에 따른 이벤트를 switch case로 구분
            switch (map[stage].tile[row][column]) {
            case 6:
                strcpy(eventMessage, ">> 6번 칸입니다. 기능 미구현 <<");
                break;
            case 5:
                strcpy(eventMessage, ">> 닫힌 문입니다! 열쇠가 필요합니다. <<");
                break;
            case 4: // 열쇠 획득 이벤트
                hasKey = 1;
                map[stage].tile[row][column] = 0;
                strcpy(eventMessage, ">> 열쇠를 획득했습니다! 닫힌 문을 엽니다. <<");
                break;
            case 3: // 시간 연장
                map[stage].timeLimit += 8;
                map[stage].tile[row][column] = 0;
                strcpy(eventMessage, ">> 시간 연장(+8초) 획득! <<");
                break;

            case 2: // 도착 지점
                printf("\033[%d;%df옷", row + 2, column * 2 + 1);
                printf("\033[%d;1f", map[stage].height + 3);
                return 1;
            default:
                strcpy(eventMessage, "");
                break;
            }

            printf("\033[%d;%df옷", row + 2, column * 2 + 1);
        }
        Sleep(40);
    }
}

void showStartScreen()
{
    if (!hasStarted) {
        printf("$$\\     $$\\ $$$$$$\\   /$$$$$$$$ /$$$$$$$$\\\n");
        printf("$$$\\   $$$ |$$  __$$\\ |_____ $$ | $$_____/\n");
        printf("$$$$\\ $$$$ |$$ /  $$ |     /$$/ | $$      \n");
        printf("$$\\$$\\$$ $$ |$$$$$$$$ |    /$$/  | $$$$$   \n");
        printf("$$ \\$$$  $$ |$$  __$$ |   /$$/   | $$__/   \n");
        printf("$$ |\\$  /$$ |$$ |  $$ |  /$$/    | $$      \n");
        printf("$$ | \\_/ $$ |$$ |  $$ | /$$$$$$$$| $$$$$$$$\\\n");
        printf("\\__|     \\__|\\__|  \\__||________/|________/\n");

        printf("\n\n       화면을 전체 화면으로 해주세요.");

        //키 눌러야 진행되도록 설정
        char start = _getch();
        

        hasStarted++;
    }

    system("cls");
    printf("  /$$$$$$\n");
    printf(" /$$__  $$\n");
    printf("|__/  \\ $$\n");
    printf("   /$$$$$/\n");
    printf("  |___  $$\n");
    printf(" /$$  \\ $$\n");
    printf("|  $$$$$$/\n");
    printf(" \\______/ \n");

    Sleep(1000);
    system("cls");

    printf("  /$$$$$$ \n");
    printf(" /$$__  $$\n");
    printf("|__/  \\ $$\n");
    printf("  /$$$$$$/\n");
    printf(" /$$____/ \n");
    printf("| $$      \n");
    printf("| $$$$$$$$\n");
    printf("|________/\n");

    Sleep(1000);
    system("cls");

    printf("   /$$  \n");
    printf(" /$$$$  \n");
    printf("|_  $$  \n");
    printf("  | $$  \n");
    printf("  | $$  \n");
    printf("  | $$  \n");
    printf(" /$$$$$$\n");
    printf("|______/\n");

    Sleep(1000);
    system("cls");

    printf("  /$$$$$$ \n");
    printf(" /$$    $$\n");
    printf("| $$    $$\n");
    printf("| $$    $$\n");
    printf("| $$    $$\n");
    printf("| $$    $$\n");
    printf("|  $$$$$$/\n");
    printf(" \\______/ \n");

    Sleep(1000);
    system("cls");
}

void showEndingScreen() {
    system("cls");

    printf("  /$$$$$$   /$$       /$$$$$$$$  /$$$$$$   /$$$$$$$   /$$\n");
    printf(" /$$__  $$ | $$      | $$_____/ /$$__  $$ | $$__  $$ | $$\n");
    printf("| $$  \\__/ | $$      | $$      | $$  \\ $$ | $$  \\ $$ | $$\n");
    printf("| $$       | $$      | $$$$$   | $$$$$$$$ | $$$$$$$/ | $$\n");
    printf("| $$       | $$      | $$__/   | $$__  $$ | $$__  $$ |__/\n");
    printf("| $$    $$ | $$      | $$      | $$  | $$ | $$  \\ $$  \n");
    printf("|  $$$$$$/ | $$$$$$$$| $$$$$$$$| $$  | $$ | $$  | $$ /$$\n");
    printf(" \\______/  |________/|________/|__/  |__/ |__/  |__/ |__/\n");

    return;

}

void showClearScreen(int stage)
{
    int i;
    system("cls");

    if (stage == 1) system("color 0C");      // 빨강
    else if (stage == 2) system("color 0E"); // 노랑
    else if (stage == 3) system("color 0B"); // 파랑

    for (i = 0; i < 10; i++) printf("\n");

    if (stage == 1) {
        printf("                  ####################################################################################\n");
        printf("                  ##                                                                                ##\n");
        printf("                  ##   ____ _____  _    ____ _____   _      ____ _     _____   _    ____  _       ##\n");
        printf("                  ##  / ___|_   _|/ \\  / ___| ____| / |    / ___| |   | ____| / \\  |  _ \\| |      ##\n");
        printf("                  ##  \\___ \\ | | / _ \\| |  _|  _|   | |   | |   | |   |  _|  / _ \\ | |_) | |      ##\n");
        printf("                  ##   ___) || |/ ___ \\ |_| | |___  | |   | |___| |___| |___/ ___ \\|  _ < |_|      ##\n");
        printf("                  ##  |____/ |_/_/   \\_\\____|_____| |_|    \\____|_____|_____/_/   \\_\\_| \\_(_)     ##\n");
        printf("                  ##                                                                                ##\n");
        printf("                  ####################################################################################\n");
    }
    else if (stage == 2) {
        printf("                  ####################################################################################\n");
        printf("                  ##                                                                                ##\n");
        printf("                  ##   ____ _____  _    ____ _____  ____     ____ _     _____   _    ____  _      ##\n");
        printf("                  ##  / ___|_   _|/ \\  / ___| ____||___ \\   / ___| |   | ____| / \\  |  _ \\| |     ##\n");
        printf("                  ##  \\___ \\ | | / _ \\| |  _|  _|    __) | | |   | |   |  _|  / _ \\ | |_) | |     ##\n");
        printf("                  ##   ___) || |/ ___ \\ |_| | |___  / __/  | |___| |___| |___/ ___ \\|  _ < |_|     ##\n");
        printf("                  ##  |____/ |_/_/   \\_\\____|_____||____|   \\____|_____|_____/_/   \\_\\_| \\_(_)    ##\n");
        printf("                  ##                                                                                ##\n");
        printf("                  ####################################################################################\n");
    }
    else {
        printf("                  ####################################################################################\n");
        printf("                  ##                                                                                ##\n");
        printf("                  ##   ____ _____  _    ____ _____  _____    ____ _     _____   _    ____  _      ##\n");
        printf("                  ##  / ___|_   _|/ \\  / ___| ____||___ /   / ___| |   | ____| / \\  |  _ \\| |     ##\n");
        printf("                  ##  \\___ \\ | | / _ \\| |  _|  _|    |_ \\   | |   | |   |  _|  / _ \\ | |_) | |     ##\n");
        printf("                  ##   ___) || |/ ___ \\ |_| | |___  ___) |  | |___| |___| |___/ ___ \\|  _ < |_|     ##\n");
        printf("                  ##  |____/ |_/_/   \\_\\____|_____|____/    \\____|_____|_____/_/   \\_\\_| \\_(_)    ##\n");
        printf("                  ##                                                                                ##\n");
        printf("                  ####################################################################################\n");
    }

    printf("\n\n                                     >> NEXT STAGE: PRESS ANY KEY <<\n");

    for (i = 0; i < 10; i++) printf("\n");

    rewind(stdin);
    system("pause > nul");

    system("color 0F");
    system("cls");

}

//스테이지 txt로 추출
void exportMapToFile(int stage)
{
    char filename[50];
    sprintf(filename, "map_export_stage_%d.txt", stage + 1);

    FILE* fp = fopen(filename, "w");
    if (fp == NULL) {
        printf("맵 파일 생성에 실패했습니다.\n");
        return;
    }

    for (int i = 0; i < map[stage].height; i++) {
        for (int j = 0; j < map[stage].width; j++) {
            fprintf(fp, "%d", map[stage].tile[i][j]);
        }
        fprintf(fp, "\n");
    }

    fclose(fp);
}

//txt파일로 만든 스테이지 읽어오기
void importMapFromFile(int stage, const char* filename, int limit, const char* name)
{
    FILE* fp = fopen(filename, "r");
    if (fp == NULL) {
        printf("맵 파일을 찾을 수 없습니다: %s\n", filename);
        return;
    }

    int h = 0;
    int w = 0;
    char buffer[200];

    while (fgets(buffer, sizeof(buffer), fp) != NULL) {
        buffer[strcspn(buffer, "\r\n")] = '\0';

        if (h == 0) {
            w = strlen(buffer);
        }

        for (int j = 0; j < w; j++) {
            map[stage].tile[h][j] = buffer[j] - '0';
        }
        h++;
    }

    fclose(fp);

    map[stage].width = w;
    map[stage].height = h;
    map[stage].timeLimit = limit;
    map[stage].startRow = 1;
    map[stage].startCol = 1;
    strcpy(map[stage].levelName, name);
}
