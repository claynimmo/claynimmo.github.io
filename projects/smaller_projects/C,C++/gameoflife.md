---
layout: default
title: Portfolio |  Conway's Game of Life
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Conway's Game of Life

The game of life is a cellular simulation, with the following rules: any cell with less than two or more than three live neighbours dies; a cell with exactly 2 or 3 neighbours lives; a dead cell with exactly three neighbours becomes alive. This program performs the simulation inside the console, using ascii characters to display the grid.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#ifdef _WIN32
#include <windows.h>
#else
#include <unistd.h>
#endif

#define gridIndex(row, col, cols) ((row) * (cols) + (col))


#define WAIT_TIME_MS 700
#define ALIVE 1
#define EMPTY 0
#define PENDING_SPAWN 2 // marker for if the cell is marked to spawn, without impacting the rest of the grid
#define PENDING_DEATH 3 // marker for if the cell is marked to die, without impacting the rest of the grid
// macros to determine the status of a cell
#define IS_ALIVE(val) (val == ALIVE || val == PENDING_DEATH)
#define IS_DEAD(val)  (val == EMPTY || val == PENDING_SPAWN)

#define LIVE_CELL_DISPLAY "██"
#define DEAD_CELL_DISPLAY "░░"
#define TXT_LIVE_CELL 'O'
#define TXT_DEAD_CELL 'X'

void sleep_ms(int ms){
    #ifdef _WIN32
        Sleep(ms);
    #else
        sleep(ms/1000);
    #endif
}
int GetNeighbor(short* grid, int gridSize, int x, int y){
    int numNeighbors = 0;
    for(int i = -1; i <= 1; i++){
        int xIndex = x + i;
        if(xIndex >= gridSize || xIndex < 0) continue;

        for(int j = -1; j <= 1; j++){
            if(i == 0 && j == 0) continue;
            int yIndex = y + j;
            if(yIndex >= gridSize || yIndex < 0) continue;

            int index = gridIndex(xIndex, yIndex, gridSize);
            if(IS_ALIVE(grid[index])){
                numNeighbors ++;
            }
        }
    }
    return numNeighbors;
}

int GetNewCellStatus(short currentVal, int numNeighbors){
    if(IS_DEAD(currentVal) && numNeighbors == 3) return PENDING_SPAWN;
    if(IS_ALIVE(currentVal) && (numNeighbors < 2 || numNeighbors > 3)) return PENDING_DEATH;
    return currentVal;
}

void PrintGrid(short* grid, int gridSize){
    printf("\033[%dA", gridSize);   // move cursor to top left, to override the terminal
    for(int y = 0; y < gridSize; y++){
        for(int x = 0; x < gridSize; x++){
            int gridIndex = gridIndex(x,y,gridSize);
            short gridValue = grid[gridIndex];
            
            if(IS_ALIVE(gridValue)) printf("%s",LIVE_CELL_DISPLAY);
            else if(IS_DEAD(gridValue)) printf("%s",DEAD_CELL_DISPLAY);
        }
        printf("\n");
    }
}

void main_loop(short* grid, int gridSize, int iterations){

    int numAlive = 0;
    int numChanged = 0;
    PrintGrid(grid,gridSize);
    sleep_ms(WAIT_TIME_MS);
    for(int i = 0; i < iterations; i++){
        // mark cell
        for(int y = 0; y < gridSize; y++){
            for(int x = 0; x < gridSize; x++){
                int gridIndex = gridIndex(x,y,gridSize);
                short gridValue = grid[gridIndex];
                
                int neighbors = GetNeighbor(grid, gridSize, x, y);
                grid[gridIndex] = GetNewCellStatus(gridValue, neighbors);
            }
        }

        // apply cells
        numAlive = 0;
        numChanged = 0;
        for(int y = 0; y < gridSize; y++){
            for(int x = 0; x < gridSize; x++){
                int gridIndex = gridIndex(x,y,gridSize);
                short gridValue = grid[gridIndex];
                if(gridValue == PENDING_DEATH) {
                    grid[gridIndex] = EMPTY;
                    numChanged++;
                }
                else if(gridValue == PENDING_SPAWN){
                    grid[gridIndex] = ALIVE;
                    numChanged ++;
                }
                if(IS_ALIVE(grid[gridIndex])) numAlive++;
            }
        }

        PrintGrid(grid, gridSize);
        sleep_ms(WAIT_TIME_MS);
        if(numAlive == 0 || numChanged == 0) return; // exit when there are no living cells
    }
}

int load_grid_from_file(const char* filename, short** outGrid, int* outSize) {
    FILE* file = fopen(filename, "r");
    if (!file) return 0;

    char buffer[1024];
    int rows = 0;
    int cols = 0;

    while (fgets(buffer, sizeof(buffer), file)) {
        int fileLength = strlen(buffer);

        if (buffer[fileLength-1] == '\n') buffer[--fileLength] = '\0';

        if (cols == 0) cols = fileLength;
        else if (fileLength != cols){ // break if the row lengths are not all the same
            fclose(file);
            return 0;
        }

        rows++;
    }

    *outGrid = malloc(sizeof(short) * rows * cols);
    *outSize = rows;

    // fill the grid
    fseek(file, 0, SEEK_SET);
    int r = 0;
    while (fgets(buffer, sizeof(buffer), file)) {
        for (int c = 0; c < cols; c++) {
            char ch = buffer[c];

            if (ch == TXT_LIVE_CELL)
                (*outGrid)[gridIndex(c, r, cols)] = ALIVE;
            else
                (*outGrid)[gridIndex(c, r, cols)] = EMPTY;
        }
        r++;
    }

    fclose(file);
    return 1;
}

int main(int argc, char *argv[]){

    #ifdef _WIN32
        SetConsoleOutputCP(CP_UTF8);
        SetConsoleCP(CP_UTF8);
    #endif

    short* grid;
    int gridSize;
    char filePath[256];
    printf("Input File Path: ");
    scanf("%s", &filePath);

    if (!load_grid_from_file(filePath, &grid, &gridSize)) {
        printf("Failed to load grid\n");
        return 1;
    }

    int iterations = 0;
    
    printf("\nInput Iterations: ");
    scanf("%d", &iterations);

    if(gridSize < 5) return 0;

    for (int i = 0; i < gridSize + 2; i++)
        printf("\n");


    main_loop(grid, gridSize, iterations);


    do{
        printf("Continue Simulation (input -1 to stop, otherwise input number of iterations): ");
        scanf("%d", &iterations);
        if(iterations >= 1){
            for (int i = 0; i < gridSize + 2; i++)
                printf("\n");
            main_loop(grid, gridSize, iterations);
        }
    } while(iterations > 0);

    free(grid);
    return 0;
}
```