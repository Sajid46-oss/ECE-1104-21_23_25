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


void viewAllStudents(void) {
    if (studentCount == 0) { printf("\nNo records yet.\n"); return; }
    printf("\n%-10s %-25s %-6s %-15s\n", "Roll", "Name", "Group", "Phone");
    printDivider();
    for (int i = 0; i < studentCount; i++) {
        printf("%-10s %-25s %-6s %-15s\n",
               students[i].roll, students[i].name, students[i].bloodGroup, students[i].phone);
    }
    printf("\nTotal: %d record(s).\n", studentCount);
}

void findByBloodGroup(void) {
    char bg[5];
    printf("\nEnter the blood group (the one you have, or the one you need): ");
    readLine(bg, sizeof(bg));
    toUpperStr(bg);
    if (!isValidBloodGroup(bg)) { printf("Invalid blood group.\n"); return; }

    int found = 0;
    printf("\nPeople with blood group %s:\n", bg);
    printf("%-10s %-25s %-15s\n", "Roll", "Name", "Phone");
    printDivider();
    for (int i = 0; i < studentCount; i++) {
        if (strcmp(students[i].bloodGroup, bg) == 0) {
            printf("%-10s %-25s %-15s\n", students[i].roll, students[i].name, students[i].phone);
            found++;
        }
    }
    if (!found) printf("No one in the records has blood group %s.\n", bg);
    else printf("\nTotal: %d people with blood group %s.\n", found, bg);
}

/* ---------------------- Update & delete ---------------------- */

void updateStudent(void) {
    char roll[15];
    printf("Enter roll of student to update: ");
    readLine(roll, sizeof(roll));
    int idx = findIndexByRoll(roll);
    if (idx == -1) { printf("Student not found.\n"); return; }

    int choice;
    do {
        printf("\n--- Update %s (Roll %s) ---\n", students[idx].name, students[idx].roll);
        printf("1. Name (%s)\n", students[idx].name);
        printf("2. Blood Group (%s)\n", students[idx].bloodGroup);
        printf("3. Phone (%s)\n", students[idx].phone);
        printf("0. Done\n");
        printf("Choice: ");
        scanf("%d", &choice);
        flushInputBuffer();

        switch (choice) {
            case 1:
                printf("New name: ");
                readLine(students[idx].name, sizeof(students[idx].name));
                break;
            case 2: {
                char bg[5];
                do {
                    printf("New blood group: ");
                    readLine(bg, sizeof(bg));
                    toUpperStr(bg);
                    if (!isValidBloodGroup(bg)) printf("Invalid group.\n");
                } while (!isValidBloodGroup(bg));
                strcpy(students[idx].bloodGroup, bg);
                break;
            }
            case 3:
                printf("New phone: ");
                readLine(students[idx].phone, sizeof(students[idx].phone));
                break;
            case 0: break;
            default: printf("Invalid choice.\n");
        }
    } while (choice != 0);

    saveStudents();
    printf("Updated. Data synced.\n");
}

void deleteStudent(void) {
    char roll[15];
    printf("Enter roll of student to delete: ");
    readLine(roll, sizeof(roll));
    int idx = findIndexByRoll(roll);
    if (idx == -1) { printf("Student not found.\n"); return; }

    printf("Delete %s (Roll %s)? (y/n): ", students[idx].name, students[idx].roll);
    char c;
    scanf(" %c", &c);
    flushInputBuffer();
    if (tolower((unsigned char) c) != 'y') { printf("Cancelled.\n"); return; }

    for (int i = idx; i < studentCount - 1; i++) students[i] = students[i + 1];
    studentCount--;
    saveStudents();
    printf("Deleted. Data synced.\n");
}

/* ---------------------- Menu ---------------------- */

void mainMenu(void) {
    int choice;
    do {
        printf("\n=====================================================\n");
        printf("   BLOOD SYNC - Department Blood Group Directory\n");
        printf("=====================================================\n");
        printf("1. Add Student(s)\n");
        printf("2. Import Students from CSV File\n");
        printf("3. View All Students\n");
        printf("4. Find People By Blood Group  <-- main feature\n");
        printf("5. Update a Student\n");
        printf("6. Delete a Student\n");
        printf("0. Exit\n");
        printf("Enter choice: ");
        scanf("%d", &choice);
        flushInputBuffer();

        switch (choice) {
            case 1: addMultipleStudents(); break;
            case 2: importFromCSV(); break;
            case 3: viewAllStudents(); break;
            case 4: findByBloodGroup(); break;
            case 5: updateStudent(); break;
            case 6: deleteStudent(); break;
            case 0: printf("Data saved. Goodbye!\n"); break;
            default: printf("Invalid choice, try again.\n");
        }
    } while (choice != 0);
}

```

## *Info File*
In this file we've kept the information of 60 students of our department .
[roster.csv](https://github.com/user-attachments/files/31750563/roster.csv)
roll,name,bloodgroup
2510001,Auritri Barua,A+
2510002,A. M. Montasir Un Nobi Borno,A+
2510003,Zobair Ibn Amin,B+
2510004,Onirban Deb,B+
2510005,Md. Samiullah,B+
2510006,Mumtahinah Munaja Tahi,B-
2510007,Tashrif Islam Sarat,A+
2510008,Shahria Hossain Tabassum,B+
2510009,Mst. Fouzia Hoque Dristy,A+
2510010,Md. Fazle Rabbi,O+
2510011,Sayon Ghosh Arnob,A+
2510012,Parthib Sutradhar,A+
2510013,Sumaiya Sultana Maksura,B+
2510014,Md. Mosabbir Salehin,B+
2510015,Umma Farzana Kabir,B+
2510016,Golam Sadat Raihan,B-
2510017,Md.Omar Faruk Shifat,A+
2510018,Tashfia Islam Saba,O+
2510019,Alif Al Hasan,A+
2510020,Md. Amir Hamza,AB+
2510021,Ahnaf Hameem,B+
2510022,Jannaty Akter,O+
2510023,MD Sajid Hasan,A+
2510024,Jahrin Taslim,O+
2510025,Md. Toufikul Islam Omi,AB+
2510026,Hasan Mushfiq,B+
2510027,Tanbina Akter Tanisha,A+
2510028,Mohammad Ahnaf Habib,B+
2510029,Md. Shahriar Mahmud Pias,A+
2510030,Tasfia Zinat,A+
2510031,K. M. Ali Imran Nur,A+
2510032,Anika Tahsin,B+
2510033,Sajib Dutta,O+
2510034,Ehsunul Islam,O+
2510035,AL-Fahim,O+
2510036,Md. Mustahid Parvez,O+
2510037,Md Rifat Rahman Deshad,A+
2510038,Tanisha Khan,O+
2510039,Fahmida Akther Ayman,O+
2510040,Sajidur Rahman,B-
2510041,Md. Muhaiminul Islam,O+
2510042,Jaber Al Masud Joy,B+
2510043,Md. Sourav Hossain Medha,AB+
2510044,Jarin Tasnim,B+
2510045,Md. Shanewaj Islam Raj,A+
2510046,Rajdip Kormokar,O+
2510047,Ahmed Ibrahim,A+
2510048,Talha Hasin Bin Haroon,A+
2510049,Mobeshir Hossain Fakir,O+
2510050,Mamun Rashid,O-
2510051,Noor Mohammod,B+
2510052,Nishika Sharif,O-
2510053,Rahul Sarker,B+
2510054,Najib Ahmed Ratul,O+
2510055,Safwan Ahmed,O-
2510056,Simanto Kumar Sarkar,O+
2510057,Mahbub Shahriar,A+
2510058,Dipto Biswas,B+
2510059,Md. Faruque Al Harun,B+
2510060,Akib Hasan Rafi,B+


## *Output*
<img width="517" height="343" alt="Screenshot 2026-09-03 001149" src="https://github.com/user-attachments/assets/a6d307cc-cb8a-4f18-a5a3-1bbad4965b0c" />
<img width="470" height="347" alt="Screenshot 2026-09-03 001215" src="https://github.com/user-attachments/assets/bbb1810d-1cfb-4cbc-9050-ed675afdb610" />


