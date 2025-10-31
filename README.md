// dino.c
// Simple terminal "Chrome Dino" clone in C
// Compile: gcc -o dino dino.c -std=c99
// Run: ./dino

#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
#include <time.h>
#include <sys/select.h>
#include <string.h>
#include <signal.h>

#define WIDTH 80
#define HEIGHT 20
#define GROUND_Y (HEIGHT-3)
#define DINO_X 6
#define MAX_OBS 6

struct termios orig_term;

void restore_terminal() {
    tcsetattr(STDIN_FILENO, TCSANOW, &orig_term);
    printf("\033[?25h"); // show cursor
}

void handle_exit(int sig) {
    (void)sig;
    restore_terminal();
    printf("\nGoodbye!\n");
    exit(0);
}

void setup_terminal() {
    struct termios t;
    tcgetattr(STDIN_FILENO, &orig_term);
    t = orig_term;
    t.c_lflag &= ~(ICANON | ECHO); // non-canonical, no echo
    t.c_cc[VMIN] = 0;
    t.c_cc[VTIME] = 0;
    tcsetattr(STDIN_FILENO, TCSANOW, &t);
    printf("\033[?25l"); // hide cursor
    atexit(restore_terminal);
    signal(SIGINT, handle_exit);
    signal(SIGTERM, handle_exit);
}

int kbhit() {
    struct timeval tv = {0L, 0L};
    fd_set fds;
    FD_ZERO(&fds);
    FD_SET(STDIN_FILENO, &fds);
    return select(STDIN_FILENO+1, &fds, NULL, NULL, &tv) > 0;
}

int getch_nonblock() {
    if (!kbhit()) return -1;
    char c;
    if (read(STDIN_FILENO, &c, 1) <= 0) return -1;
    return (int)c;
}

typedef struct {
    int x;
    int type; // 0 = small cactus, 1 = tall cactus
    int active;
} Obstacle;

void clear_screen() {
    printf("\033[H\033[J");
}

void draw_frame(int dino_y, Obstacle obs[], int score) {
    clear_screen();
    char screen[HEIGHT][WIDTH+1];
    for (int r=0;r<HEIGHT;r++){
        for (int c=0;c<WIDTH;c++) screen[r][c] = ' ';
        screen[r][WIDTH] = '\0';
    }

    // ground
    for (int c=0;c<WIDTH;c++) {
        screen[GROUND_Y+1][c] = '_';
    }

    // draw dino (2x2-ish)
    int dy = dino_y;
    if (dy < 0) dy = 0;
    if (dy > GROUND_Y-1) dy = GROUND_Y-1;
    screen[dy][DINO_X] = 'O';
    screen[dy+1][DINO_X] = '/';
    screen[dy+1][DINO_X+1] = '\\';

    // obstacles
    for (int i=0;i<MAX_OBS;i++){
        if (!obs[i].active) continue;
        int x = obs[i].x;
        if (x >= 0 && x < WIDTH) {
            if (obs[i].type == 0) {
                // small cactus (1 height)
                screen[GROUND_Y][x] = '|';
            } else {
                // tall cactus (2 height)
                screen[GROUND_Y-1][x] = '|';
                screen[GROUND_Y][x]   = '|';
            }
        }
    }

    // print screen
    for (int r=0;r<HEIGHT;r++){
        printf("%s\n", screen[r]);
    }
    printf("\nScore: %d    (Space to jump, q to quit)\n", score);
    fflush(stdout);
}

int check_collision(int dino_y, Obstacle obs[]) {
    for (int i=0;i<MAX_OBS;i++){
        if (!obs[i].active) continue;
        int x = obs[i].x;
        if (x == DINO_X || x == DINO_X+1) {
            // check overlap in vertical
            if (obs[i].type == 0) {
                // occupies GROUND_Y
                if (dino_y+1 >= GROUND_Y) return 1;
            } else {
                if (dino_y+1 >= GROUND_Y-1) return 1;
            }
        }
    }
    return 0;
}

int main() {
    srand((unsigned)time(NULL));
    setup_terminal();

    Obstacle obs[MAX_OBS];
    for (int i=0;i<MAX_OBS;i++) { obs[i].active = 0; obs[i].x = 0; obs[i].type = 0; }

    int dino_y = GROUND_Y-1; // top row index for dino head
    double vel = 0.0;
    const double GRAV = 0.9;
    const double JUMP_V = -12.0;

    int spawn_timer = 0;
    int speed = 1; // obstacle speed (cells per frame)
    int score = 0;
    int frame = 0;
    int highscore = 0;

    // main loop ~50 FPS
    const useconds_t FRAME_USEC = 20000; // 20ms -> 50 fps

    while (1) {
        int ch = getch_nonblock();
        if (ch == 'q' || ch == 'Q') break;
        if (ch == ' ' || ch == 'w' || ch == 'W') {
            // jump if on ground
            if (dino_y >= GROUND_Y-1) {
                vel = JUMP_V;
            }
        }

        // update dino physics
        vel += GRAV;
        dino_y += (int)vel;
        if (dino_y >= GROUND_Y-1) {
            dino_y = GROUND_Y-1;
            vel = 0;
        }
        if (dino_y < 0) {
            dino_y = 0;
            vel = 0;
        }

        // spawn obstacles occasionally
        if (spawn_timer <= 0) {
            // find an inactive slot
            for (int i=0;i<MAX_OBS;i++){
                if (!obs[i].active) {
                    obs[i].active = 1;
                    obs[i].x = WIDTH - 2;
                    obs[i].type = (rand()%5 == 0) ? 1 : 0; // some tall ones
                    break;
                }
            }
            spawn_timer = 20 + rand()%60; // frames until next spawn
        } else {
            spawn_timer--;
        }

        // move obstacles
        for (int i=0;i<MAX_OBS;i++){
            if (!obs[i].active) continue;
            obs[i].x -= speed;
            if (obs[i].x < -2) obs[i].active = 0;
        }

        // collision
        if (check_collision(dino_y, obs)) {
            // game over
            clear_screen();
            printf("GAME OVER! Score: %d\n", score);
            if (score > highscore) highscore = score;
            printf("Highscore: %d\n", highscore);
            printf("Press 'r' to restart, 'q' to quit.\n");
            fflush(stdout);
            // wait for choice
            int choice = -1;
            while (1) {
                int c = getch_nonblock();
                if (c == 'q' || c == 'Q') { restore_terminal(); printf("\nGoodbye!\n"); return 0; }
                if (c == 'r' || c == 'R') break;
                usleep(100000);
            }
            // reset game state
            for (int i=0;i<MAX_OBS;i++) obs[i].active = 0;
            dino_y = GROUND_Y-1;
            vel = 0;
            spawn_timer = 0;
            score = 0;
            frame = 0;
            speed = 1;
            continue;
        }

        // increase difficulty slowly
        frame++;
        if (frame % 200 == 0) {
            if (speed < 4) speed++;
        }

        // increment score
        score += 1;

        // draw
        draw_frame(dino_y, obs, score);

        usleep(FRAME_USEC);
    }

    restore_terminal();
    printf("\nExited. Final score: %d\n", score);
    return 0;
}
