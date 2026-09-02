## *Code*
```c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_STUDENTS 100      /* headroom above ~60 department members */
#define NUM_GROUPS   8
#define STUDENT_FILE "students.dat"

typedef struct {
    char roll[15];
    char name[50];
    char bloodGroup[5];
    char phone[15];
} Student;

Student students[MAX_STUDENTS];
int studentCount = 0;

const char *bloodGroups[NUM_GROUPS] =
    {"O-", "O+", "A-", "A+", "B-", "B+", "AB-", "AB+"};

/* ---------------------- Prototypes ---------------------- */

void flushInputBuffer(void);
void readLine(char *buffer, int size);
void toUpperStr(char *s);
void trim(char *s);
void printDivider(void);
int  isValidBloodGroup(const char *group);
int  findIndexByRoll(const char *roll);

void loadStudents(void);
void saveStudents(void);

void addStudentEntry(void);
void addMultipleStudents(void);
void importFromCSV(void);
void viewAllStudents(void);
void findByBloodGroup(void);
void updateStudent(void);
void deleteStudent(void);

void mainMenu(void);

/* ---------------------- main ---------------------- */

int main(void) {
    loadStudents();
    printf("Blood Sync - Department Directory started. Loaded %d record(s).\n", studentCount);
    mainMenu();
    return 0;
}

/* ---------------------- Utility functions ---------------------- */

void flushInputBuffer(void) {
    int c;
    while ((c = getchar()) != '\n' && c != EOF) { }
}

void readLine(char *buffer, int size) {
    if (fgets(buffer, size, stdin)) {
        buffer[strcspn(buffer, "\n")] = '\0';
    } else {
        buffer[0] = '\0';
    }
    trim(buffer);
}

void toUpperStr(char *s) {
    for (int i = 0; s[i]; i++) s[i] = (char) toupper((unsigned char) s[i]);
}

void trim(char *s) {
    int start = 0;
    while (s[start] == ' ' || s[start] == '\t') start++;
    if (start > 0) memmove(s, s + start, strlen(s + start) + 1);
    int len = (int) strlen(s);
    while (len > 0 && (s[len - 1] == ' ' || s[len - 1] == '\t')) { s[len - 1] = '\0'; len--; }
}

void printDivider(void) {
    printf("--------------------------------------------------------------\n");
}

int isValidBloodGroup(const char *group) {
    for (int i = 0; i < NUM_GROUPS; i++)
        if (strcmp(bloodGroups[i], group) == 0) return 1;
    return 0;
}

int findIndexByRoll(const char *roll) {
    for (int i = 0; i < studentCount; i++)
        if (strcmp(students[i].roll, roll) == 0) return i;
    return -1;
}

/* ---------------------- File persistence ---------------------- */

void loadStudents(void) {
    FILE *fp = fopen(STUDENT_FILE, "rb");
    if (!fp) { studentCount = 0; return; }
    if (fread(&studentCount, sizeof(int), 1, fp) != 1) studentCount = 0;
    if (studentCount > MAX_STUDENTS) studentCount = MAX_STUDENTS;
    if (studentCount > 0) fread(students, sizeof(Student), (size_t) studentCount, fp);
    fclose(fp);
}
