
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#ifdef _WIN32
#include <windows.h>
#include <conio.h>
#define SLEEP(ms) Sleep(ms)
#else
#include <unistd.h>
#include <termios.h>
#include <fcntl.h>
#define SLEEP(ms) usleep((ms) * 1000)

static struct termios orig_termios;

void reset_terminal_mode(void) {
    tcsetattr(STDIN_FILENO, TCSANOW, &orig_termios);
}

void init_keyboard(void) {
    struct termios new_termios;
    tcgetattr(STDIN_FILENO, &orig_termios);
    atexit(reset_terminal_mode);
    
    new_termios = orig_termios;
    new_termios.c_lflag &= ~(ICANON | ECHO);
    new_termios.c_cc[VMIN] = 0;
    new_termios.c_cc[VTIME] = 0;
    tcsetattr(STDIN_FILENO, TCSANOW, &new_termios);
}

int kbhit(void) {
    char ch;
    int nread = read(STDIN_FILENO, &ch, 1);
    if (nread > 0) {
        ungetc(ch, stdin);
        return 1;
    }
    return 0;
}

char getch(void) {
    char ch = 0;
    if (read(STDIN_FILENO, &ch, 1) < 0) return 0;
    return ch;
}
#endif

#define WIDTH 24
#define HEIGHT 18
#define HIGH_SCORE_FILE "highscore.dat"

typedef struct {
    int x;
    int y;
    int shield; // 1 if active, 0 otherwise
} Player;

typedef struct {
    int x;
    int y;
    int type; // 0 = Obstacle (X), 1 = Bonus Coin ($)
    int active;
} Entity;

typedef struct {
    Player player;
    Entity entities[HEIGHT];
    int score;
    int highScores[3]; // Top 3 scores
    int speed;
    int gameOver;
} Game;

void resetCursor() {
    printf("\033[H");
}

void loadHighScores(int scores[3]) {
    FILE *f = fopen(HIGH_SCORE_FILE, "r");
    if (f) {
        for (int i = 0; i < 3; i++) {
            if (fscanf(f, "%d", &scores[i]) != 1) {
                scores[i] = 0;
            }
        }
        fclose(f);
    } else {
        for (int i = 0; i < 3; i++) scores[i] = 0;
    }
}

void saveHighScores(int scores[3]) {
    FILE *f = fopen(HIGH_SCORE_FILE, "w");
    if (f) {
        for (int i = 0; i < 3; i++) {
            fprintf(f, "%d\n", scores[i]);
        }
        fclose(f);
    }
}

void updateHighScores(int scores[3], int newScore) {
    for (int i = 0; i < 3; i++) {
        if (newScore > scores[i]) {
            // Shift lower scores down
            for (int j = 2; j > i; j--) {
                scores[j] = scores[j - 1];
            }
            scores[i] = newScore;
            break;
        }
    }
    saveHighScores(scores);
}

void initGame(Game *g) {
    g->player.x = WIDTH / 2;
    g->player.y = HEIGHT - 1;
    g->player.shield = 0;
    g->score = 0;
    g->speed = 70;
    g->gameOver = 0;

    for (int i = 0; i < HEIGHT; i++) {
        g->entities[i].active = 0;
    }
    loadHighScores(g->highScores);
}

void handleInput(Game *g) {
#ifdef _WIN32
    while (_kbhit()) {
        char ch = _getch();
        if ((ch == 'a' || ch == 'A' || ch == 75) && g->player.x > 1) g->player.x--;
        if ((ch == 'd' || ch == 'D' || ch == 77) && g->player.x < WIDTH - 2) g->player.x++;
        if (ch == 'q' || ch == 'Q') g->gameOver = 1;
    }
#else
    while (kbhit()) {
        char ch = getch();
        if ((ch == 'a' || ch == 'A') && g->player.x > 1) g->player.x--;
        if ((ch == 'd' || ch == 'D') && g->player.x < WIDTH - 2) g->player.x++;
        if (ch == 'q' || ch == 'Q') g->gameOver = 1;
    }
#endif
}

void updateGame(Game *g) {
    // Shift entities down
    for (int i = HEIGHT - 1; i > 0; i--) {
        g->entities[i] = g->entities[i - 1];
    }

    // Spawn new entity at row 0
    if (rand() % 4 == 0) {
        g->entities[0].active = 1;
        g->entities[0].x = 1 + (rand() % (WIDTH - 2));
        // 20% chance for a bonus coin ($), 80% for hazard (X)
        g->entities[0].type = (rand() % 5 == 0) ? 1 : 0;
    } else {
        g->entities[0].active = 0;
    }

    // Collision detection on player row
    if (g->entities[HEIGHT - 1].active && g->entities[HEIGHT - 1].x == g->player.x) {
        if (g->entities[HEIGHT - 1].type == 1) {
            // Collected Coin
            g->score += 50;
            g->entities[HEIGHT - 1].active = 0; // Clear it
        } else {
            // Hit Hazard
            if (g->player.shield > 0) {
                g->player.shield = 0; // Shield absorbs hit
                g->entities[HEIGHT - 1].active = 0;
            } else {
                g->gameOver = 1;
            }
        }
    }

    g->score += 5;
    if (g->speed > 35 && g->score % 150 == 0) {
        g->speed -= 3; // Progressive difficulty scaling
    }
}

void renderGame(Game *g) {
    resetCursor();
    printf("========== ADVANCED RACER ==========\n");
    printf("Controls: A/D = Move | Q = Quit    \n");
    printf("Score: %-5d | High Score: %-5d   \n", g->score, g->highScores[0]);
    printf("Speed: %-3dms | Shield: %s        \n\n", g->speed, g->player.shield ? "ON " : "OFF");

    for (int i = 0; i < HEIGHT; i++) {
        printf("|");
        for (int j = 0; j < WIDTH; j++) {
            if (i == g->player.y && j == g->player.x) {
                printf(g->player.shield ? "O" : "A");
            } else if (g->entities[i].active && g->entities[i].x == j) {
                printf(g->entities[i].type == 1 ? "$" : "X");
            } else {
                printf(" ");
            }
        }
        printf("|\n");
    }
    printf("------------------------------------\n");
}

int main() {
#ifndef _WIN32
    init_keyboard();
#endif

#ifdef _WIN32
    system("cls");
#else
    printf("\033[H\033[J");
#endif

    srand((unsigned int)time(NULL));
    char playAgain = 'y';

    while (playAgain == 'y' || playAgain == 'Y') {
        Game game;
        initGame(&game);

        while (!game.gameOver) {
            handleInput(&game);
            if (game.gameOver) break;

            updateGame(&game);
            renderGame(&game);

            if (game.gameOver) break;
            SLEEP(game.speed);
        }

        updateHighScores(game.highScores, game.score);

#ifndef _WIN32
        tcsetattr(STDIN_FILENO, TCSANOW, &orig_termios);
#endif

        printf("\nCRASH! Game Over. Final Score: %d\n", game.score);
        printf("Top Scores: 1st: %d | 2nd: %d | 3rd: %d\n", game.highScores[0], game.highScores[1], game.highScores[2]);
        printf("\nPlay again? (y/n): ");
        scanf(" %c", &playAgain);

#ifndef _WIN32
        init_keyboard();
#endif
    }

    printf("Thanks for playing!\n");
    return 0;
}
