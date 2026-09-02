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

void saveStudents(void) {
    FILE *fp = fopen(STUDENT_FILE, "wb");
    if (!fp) { printf("Error: could not save data!\n"); return; }
    fwrite(&studentCount, sizeof(int), 1, fp);
    if (studentCount > 0) fwrite(students, sizeof(Student), (size_t) studentCount, fp);
    fclose(fp);
}

/* ---------------------- Data entry ---------------------- */

void addStudentEntry(void) {
    if (studentCount >= MAX_STUDENTS) {
        printf("Record list is full (max %d). Increase MAX_STUDENTS in the code if needed.\n", MAX_STUDENTS);
        return;
    }

    Student s;
    printf("Roll: ");
    readLine(s.roll, sizeof(s.roll));
    if (s.roll[0] == '\0') { printf("Roll cannot be empty. Skipped.\n"); return; }
    if (findIndexByRoll(s.roll) != -1) {
        printf("A student with roll %s already exists. Skipped.\n", s.roll);
        return;
    }

    printf("Name: ");
    readLine(s.name, sizeof(s.name));

    char bg[5];
    do {
        printf("Blood Group (O-, O+, A-, A+, B-, B+, AB-, AB+): ");
        readLine(bg, sizeof(bg));
        toUpperStr(bg);
        if (!isValidBloodGroup(bg)) printf("Invalid blood group. Try again.\n");
    } while (!isValidBloodGroup(bg));
    strcpy(s.bloodGroup, bg);

    printf("Phone: ");
    readLine(s.phone, sizeof(s.phone));

    students[studentCount++] = s;
    saveStudents();
    printf("Added %s (Roll %s). [%d/%d records saved]\n", s.name, s.roll, studentCount, MAX_STUDENTS);
}

void addMultipleStudents(void) {
    printf("How many students do you want to add? (enter 1 for a single student): ");
    int n;
    scanf("%d", &n);
    flushInputBuffer();
    if (n <= 0) { printf("Nothing to add.\n"); return; }

    for (int i = 0; i < n; i++) {
        if (studentCount >= MAX_STUDENTS) {
            printf("Reached max capacity (%d). Stopping.\n", MAX_STUDENTS);
            break;
        }
        printf("\n--- Student %d of %d ---\n", i + 1, n);
        addStudentEntry();
    }
    printf("\nBulk entry finished. Total records now: %d\n", studentCount);
}

void importFromCSV(void) {
    char filename[100];
    printf("Enter CSV file name (e.g. roster.csv): ");
    readLine(filename, sizeof(filename));

    FILE *fp = fopen(filename, "r");
    if (!fp) { printf("Could not open file '%s'.\n", filename); return; }

    printf("Does the file have a header row? (y/n): ");
    char h;
    scanf(" %c", &h);
    flushInputBuffer();

    char line[300];
    int lineNum = 0, imported = 0, skipped = 0;

    while (fgets(line, sizeof(line), fp)) {
        lineNum++;
        line[strcspn(line, "\r\n")] = '\0';
        if (line[0] == '\0') continue;
        if (lineNum == 1 && tolower((unsigned char) h) == 'y') continue;

        if (studentCount >= MAX_STUDENTS) {
            printf("Reached max capacity (%d). Stopping import.\n", MAX_STUDENTS);
            break;
        }

        Student s;
        memset(&s, 0, sizeof(s));
        char *tok;

        tok = strtok(line, ","); if (!tok) { skipped++; continue; }
        trim(tok); strncpy(s.roll, tok, sizeof(s.roll) - 1);

        tok = strtok(NULL, ","); if (!tok) { skipped++; continue; }
        trim(tok); strncpy(s.name, tok, sizeof(s.name) - 1);

        tok = strtok(NULL, ","); if (!tok) { skipped++; continue; }
        trim(tok); toUpperStr(tok);
        if (!isValidBloodGroup(tok)) { skipped++; continue; }
        strcpy(s.bloodGroup, tok);

        tok = strtok(NULL, ",");
        if (tok) { trim(tok); strncpy(s.phone, tok, sizeof(s.phone) - 1); }
        else { s.phone[0] = '\0'; } /* phone is optional - a roster without it still imports */

        if (s.roll[0] == '\0' || findIndexByRoll(s.roll) != -1) { skipped++; continue; }

        students[studentCount++] = s;
        imported++;
    }
    fclose(fp);
    saveStudents();
    printf("Import complete: %d added, %d skipped (bad format or duplicate roll). Total: %d\n",
           imported, skipped, studentCount);
}

/* ---------------------- Viewing & searching ---------------------- */
