#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define FILE_NAME "tasks.txt"
#define NAME_LEN 100

typedef struct Task {
    int id;
    char name[NAME_LEN];
    struct Task *next;
} Task;

Task *head = NULL;
int nextId = 1;

/* -------- File helpers -------- */

/* Append one task to file (used after add) */
void appendTaskToFile(const Task *t) {
    FILE *fp = fopen(FILE_NAME, "a");
    if (!fp) {
        perror("Failed to open tasks file for appending");
        return;
    }
    /* Write as: id<TAB>name\n */
    fprintf(fp, "%d\t%s\n", t->id, t->name);
    fclose(fp);
}

/* Rewrite entire list to file (used after delete or clear) */
void saveAllTasksToFile() {
    FILE *fp = fopen(FILE_NAME, "w");
    if (!fp) {
        perror("Failed to open tasks file for writing");
        return;
    }
    Task *cur = head;
    while (cur) {
        fprintf(fp, "%d\t%s\n", cur->id, cur->name);
        cur = cur->next;
    }
    fclose(fp);
}

/* Load tasks from file into linked list; also sets nextId */
void loadTasksFromFile() {
    FILE *fp = fopen(FILE_NAME, "r");
    if (!fp) {
        /* No file yet � that's fine */
        return;
    }

    char line[NAME_LEN + 32];
    int maxId = 0;

    while (fgets(line, sizeof(line), fp)) {
        /* Each line: id<TAB>name\n */
        char *tab = strchr(line, '\t');
        if (!tab) continue;
        *tab = '\0';
        int id = atoi(line);
        char *name = tab + 1;
        name[strcspn(name, "\r\n")] = '\0'; /* trim newline */

        /* create node */
        Task *t = (Task*)malloc(sizeof(Task));
        if (!t) { perror("malloc"); fclose(fp); return; }
        t->id = id;
        strncpy(t->name, name, NAME_LEN-1);
        t->name[NAME_LEN-1] = '\0';
        t->next = NULL;

        /* append to linked list */
        if (!head) head = t;
        else {
            Task *cur = head;
            while (cur->next) cur = cur->next;
            cur->next = t;
        }

        if (id > maxId) maxId = id;
    }

    nextId = maxId + 1;
    fclose(fp);
}

/* -------- Linked list helpers -------- */

Task* createTaskNode(const char *name) {
    Task *t = (Task*)malloc(sizeof(Task));
    if (!t) {
        perror("Memory allocation failed");
        exit(1);
    }
    t->id = nextId++;
    strncpy(t->name, name, NAME_LEN-1);
    t->name[NAME_LEN-1] = '\0';
    t->next = NULL;
    return t;
}

void addTask(const char *name) {
    Task *t = createTaskNode(name);
    if (!head) head = t;
    else {
        Task *cur = head;
        while (cur->next) cur = cur->next;
        cur->next = t;
    }
    appendTaskToFile(t); /* persist immediately */
    printf("Task added: #%d - %s\n", t->id, t->name);
}

void removeTask(int id) {
    Task *cur = head, *prev = NULL;
    while (cur) {
        if (cur->id == id) {
            if (!prev) head = cur->next;
            else prev->next = cur->next;
            printf("Removed task #%d - %s\n", cur->id, cur->name);
            free(cur);
            saveAllTasksToFile(); /* rewrite file to reflect deletion */
            return;
        }
        prev = cur;
        cur = cur->next;
    }
    printf("Task #%d not found.\n", id);
}

void displayTasks() {
    if (!head) {
        printf("No tasks available.\n");
        return;
    }
    printf("\n--- Task List ---\n");
    Task *cur = head;
    while (cur) {
        printf("Task #%d: %s\n", cur->id, cur->name);
        cur = cur->next;
    }
}

void freeAll() {
    Task *cur = head;
    while (cur) {
        Task *tmp = cur;
        cur = cur->next;
        free(tmp);
    }
    head = NULL;
    /* Optionally clear the file on exit � comment out if you want to keep tasks */
    /* FILE *fp = fopen(FILE_NAME, "w"); if (fp) fclose(fp); */
}

/* -------- Utility -------- */

/* Trim leading/trailing spaces (in-place) */
void trim(char *s) {
    /* trim leading */
    char *start = s;
    while (*start && (*start == ' ' || *start == '\t')) start++;
    if (start != s) memmove(s, start, strlen(start) + 1);
    /* trim trailing */
    size_t len = strlen(s);
    while (len > 0 && (s[len-1] == ' ' || s[len-1] == '\t' || s[len-1] == '\r' || s[len-1] == '\n')) {
        s[len-1] = '\0';
        len--;
    }
}

/* -------- Main (UI) -------- */

int main() {
    loadTasksFromFile(); /* load existing tasks, if any */

    int choice = -1;
    int id;
    char name[NAME_LEN];

    while (1) {
        printf("\n--- Linked List Task Manager (persistent) ---\n");
        printf("1. Add Task\n");
        printf("2. Remove Task\n");
        printf("3. Display Tasks\n");
        printf("0. Exit\n");
        printf("Enter choice: ");
        if (scanf("%d", &choice) != 1) {
            /* invalid input: clear stdin and continue */
            int c; while ((c = getchar()) != '\n' && c != EOF) {}
            printf("Invalid input.\n");
            continue;
        }
        getchar(); /* consume newline */

        switch (choice) {
            case 1:
                printf("Enter task name: ");
                if (!fgets(name, sizeof(name), stdin)) { printf("Input error.\n"); break; }
                name[strcspn(name, "\r\n")] = '\0';
                trim(name);
                if (strlen(name) == 0) { printf("Empty name - cancelled.\n"); break; }
                addTask(name);
                break;

            case 2:
                printf("Enter task ID to remove: ");
                if (scanf("%d", &id) != 1) {
                    int c; while ((c = getchar()) != '\n' && c != EOF) {}
                    printf("Invalid ID.\n");
                    break;
                }
                getchar(); /* consume newline */
                removeTask(id);
                break;

            case 3:
                displayTasks();
                break;

            case 0:
                freeAll();
                printf("Exiting...\n");
                return 0;

            default:
                printf("Invalid choice!\n");
        }
    }
    return 0;
}


