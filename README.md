#include <windows.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void passOne();
void passTwo();
void displayPassOne(HWND hwnd);
void displayPassTwo(HWND hwnd);
void displayObjectCode(HWND hwnd);
LRESULT CALLBACK WindowProcedure(HWND, UINT, WPARAM, LPARAM);
void AddControls(HWND);

HWND hOutputBox;
char outputBuffer[4096];

int WINAPI WinMain(HINSTANCE hInst, HINSTANCE hPrevInst, LPSTR args, int nCmdShow)
{
    WNDCLASSW wc = {0};
    wc.hbrBackground = (HBRUSH)COLOR_WINDOW;
    wc.hCursor = LoadCursor(NULL, IDC_ARROW);
    wc.hInstance = hInst;
    wc.lpszClassName = L"myWindowClass";
    wc.lpfnWndProc = WindowProcedure;
    if (!RegisterClassW(&wc)) return -1;

    CreateWindowW(L"myWindowClass", L"SIC Assembler", WS_OVERLAPPEDWINDOW | WS_VISIBLE, 100, 100, 800, 600, NULL, NULL, NULL, NULL);

    MSG msg = {0};
    while (GetMessage(&msg, NULL, NULL, NULL))
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    return 0;
}

LRESULT CALLBACK WindowProcedure(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp)
{
    switch (msg)
    {
    case WM_COMMAND:
        if (wp == 1) passOne();
        else if (wp == 2) passTwo();
        else if (wp == 3) displayPassOne(hwnd);
        else if (wp == 4) displayPassTwo(hwnd);
        else if (wp == 5) displayObjectCode(hwnd);
        break;
    case WM_CREATE:
        AddControls(hwnd);
        break;
    case WM_DESTROY:
        PostQuitMessage(0);
        break;
    default:
        return DefWindowProcW(hwnd, msg, wp, lp);
    }
    return 0;
}
void AddControls(HWND hwnd)
{
    CreateWindowW(L"Button", L"Run Pass 1", WS_VISIBLE | WS_CHILD, 50, 50, 150, 50, hwnd, (HMENU)1, NULL, NULL);
    CreateWindowW(L"Button", L"Run Pass 2", WS_VISIBLE | WS_CHILD, 220, 50, 150, 50, hwnd, (HMENU)2, NULL, NULL);
    CreateWindowW(L"Button", L"Display Pass 1", WS_VISIBLE | WS_CHILD, 50, 120, 150, 50, hwnd, (HMENU)3, NULL, NULL);
    CreateWindowW(L"Button", L"passtwo address", WS_VISIBLE | WS_CHILD, 220, 120, 150, 50, hwnd, (HMENU)4, NULL, NULL);
    CreateWindowW(L"Button", L"Display Pass 2 object code", WS_VISIBLE | WS_CHILD, 390, 50, 200, 50, hwnd, (HMENU)5, NULL, NULL);
    hOutputBox = CreateWindowW(L"Edit", L"", WS_VISIBLE | WS_CHILD | WS_BORDER | ES_MULTILINE | ES_AUTOVSCROLL | ES_READONLY, 50, 190, 700, 300, hwnd, NULL, NULL, NULL);
}


void passOne()
{
    FILE *inputFile = fopen("input.txt", "r"), *symtabFile = fopen("symtab.txt", "w"), *intermediateFile = fopen("intermediate.txt", "w");
    if (!inputFile || !symtabFile || !intermediateFile)
    {
        MessageBox(NULL, "Error opening files", "Error", MB_OK | MB_ICONERROR);
        return;
    }
    int locctr, start;
    char label[10], opcode[10], operand[10];
    fscanf(inputFile, "%s %s %s", label, opcode, operand);
    locctr = (strcmp(opcode, "START") == 0) ? strtol(operand, NULL, 16) : 0;
    start = locctr;
    while (strcmp(opcode, "END") != 0)
    {
        locctr += 3;
        if (strcmp(label, "") != 0) fprintf(symtabFile, "%s %X\n", label, locctr);
        fprintf(intermediateFile, "%04X\t%s\t%s\t%s\n", locctr, label, opcode, operand);
        fscanf(inputFile, "%s %s %s", label, opcode, operand);
    }

    fclose(inputFile);
    fclose(symtabFile);
    fclose(intermediateFile);
    MessageBox(NULL, "Pass 1 completed", "Information", MB_OK | MB_ICONINFORMATION);
}
void passTwo()
{
    FILE *intermediateFile = fopen("intermediate.txt", "r"), *symtabFile = fopen("symtab.txt", "r"), *objFile = fopen("objcode.txt", "w");
    if (!intermediateFile || !symtabFile || !objFile)
    {
        MessageBox(NULL, "Error opening files", "Error", MB_OK | MB_ICONERROR);
        return;
    }

    int address, symbolAddress, startAddress = 0, programLength = 0;
    char label[10], opcode[10], operand[10];

    fscanf(intermediateFile, "%X %s %s %s", &startAddress, label, opcode, operand);
    address = startAddress;
    programLength += 3;

    fprintf(objFile, "H^%06X^%06X\n", startAddress, programLength);

    do
    {
        if (strcmp(opcode, "BYTE") == 0)
        {
            fprintf(objFile, "T^%06X^%02X^%s\n", address, strlen(operand) - 3, operand + 2);
            address += 3;
        }
        else if (strcmp(opcode, "WORD") == 0)
        {
            fprintf(objFile, "T^%06X^03^%06X\n", address, atoi(operand));
            address += 3;
        }
        else
        {
            rewind(symtabFile);
            while (fscanf(symtabFile, "%s %X", label, &symbolAddress) != EOF)
            {
                if (strcmp(operand, label) == 0)
                {
                    fprintf(objFile, "T^%06X^03^%06X\n", address, symbolAddress);
                    address += 3;
                    break;
                }
            }
        }
    }
    while (fscanf(intermediateFile, "%X %s %s %s", &address, label, opcode, operand) != EOF);

    fprintf(objFile, "E^%06X\n", startAddress);

    fclose(intermediateFile);
    fclose(symtabFile);
    fclose(objFile);
    MessageBox(NULL, "Pass 2 completed", "Information", MB_OK | MB_ICONINFORMATION);
}

void displayObjectCode(HWND hwnd)
{
    FILE *intermediateFile, *objFile;
    strcpy(outputBuffer, "Pass 2: Input and Object Code:\r\n");
    strcat(outputBuffer, "-------------------------------------------------------------\r\n");
    strcat(outputBuffer, "Input (Address   Label   Opcode   Operand)      Object Code\r\n");
    strcat(outputBuffer, "-------------------------------------------------------------\r\n");

    intermediateFile = fopen("intermediate.txt", "r");
    objFile = fopen("objcode.txt", "r");

    if (intermediateFile && objFile)
    {
        char interAddress[10], label[10], opcode[10], operand[10];
        char objLine[100], objCode[20];

        while (fscanf(intermediateFile, "%s %s %s %s", interAddress, label, opcode, operand) != EOF)
        {


            if (strcmp(opcode, "RESW") == 0 || strcmp(opcode, "BYTE") == 0 || strcmp(opcode, "RESB") == 0)
            {

                char line[200];
                snprintf(line, sizeof(line), "%-10s %-8s %-8s %-10s      %s\r\n",
                         interAddress, label, opcode, operand, "No Obj Code");
                strcat(outputBuffer, line);
            }

            else if (fgets(objLine, sizeof(objLine), objFile))
            {
                if (sscanf(objLine, "T^%*6s^%*2s^%s", objCode) == 1)
                {

                    char line[200];
                    snprintf(line, sizeof(line), "%-10s %-8s %-8s %-10s      %s\r\n",
                             interAddress, label, opcode, operand, objCode);
                    strcat(outputBuffer, line);
                }
                else
                {

                    char line[200];
                    snprintf(line, sizeof(line), "%-10s %-8s %-8s %-10s      %s\r\n",
                             interAddress, label, opcode, operand, "Invalid Obj Code");
                    strcat(outputBuffer, line);
                }
            }
            else
            {

                char line[200];
                snprintf(line, sizeof(line), "%-10s %-8s %-8s %-10s      %s\r\n",
                         interAddress, label, opcode, operand, "No Obj Code");
                strcat(outputBuffer, line);
            }
        }

        fclose(intermediateFile);
        fclose(objFile);
    }
    else
    {
        if (!intermediateFile)
        {
            strcat(outputBuffer, "Error: intermediate.txt not found.\r\n");
        }
        if (!objFile)
        {
            strcat(outputBuffer, "Error: objcode.txt not found.\r\n");
        }
    }

    SetWindowText(hOutputBox, outputBuffer);
}

void displayPassOne(HWND hwnd)
{
    FILE *file;
    strcpy(outputBuffer, "Pass 1: Intermediate Table:\r\n");
    strcat(outputBuffer, "Address   Label       Opcode     Operand\r\n");
    strcat(outputBuffer, "----------------------------------------\r\n");

    file = fopen("intermediate.txt", "r");
    if (file)
    {
        char address[10], label[10], opcode[10], operand[10];
        while (fscanf(file, "%s %s %s %s", address, label, opcode, operand) != EOF)
        {
            char line[100];
            snprintf(line, sizeof(line), "%-10s %-10s %-10s %-10s\r\n", address, label, opcode, operand);
            strcat(outputBuffer, line);
        }
        fclose(file);
    }
    else
    {
        strcat(outputBuffer, "Error: intermediate.txt not found.\r\n");
    }

    SetWindowText(hOutputBox, outputBuffer);
}

void displayPassTwo(HWND hwnd)
{
    FILE *file;
    strcpy(outputBuffer, "Pass 2: Object Code:\r\n");
    strcat(outputBuffer, "Object Code\r\n");
    strcat(outputBuffer, "--------------------------\r\n");

    file = fopen("objcode.txt", "r");
    if (file)
    {
        char line[100];
        while (fgets(line, sizeof(line), file))
        {
            strcat(outputBuffer, line);
        }
        fclose(file);
    }
    else
    {
        strcat(outputBuffer, "Error: objcode.txt not found.\r\n");
    }

    SetWindowText(hOutputBox, outputBuffer);
}





