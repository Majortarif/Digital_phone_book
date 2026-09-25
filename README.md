# Digital_phone_book
Modernized Digital Phonebook 2026 (C console app). Contact CRUD + block, favorites, statistics &amp; export. Color UI + animations. Polishing my old 2022 project with proper fixes.
/* =======================================================================
   DIGITAL PHONEBOOK 2026
   Modernized version of the original 2022 console phonebook project.
   Windows console app (Code::Blocks / MinGW GCC).

   What changed from the 2022 version:
     - Colorized console UI (Windows console colors)
     - Typewriter title animation + spinner/loading-bar animations
     - New features: Favorite contacts, Contact statistics dashboard,
       Export contacts to a readable .txt file
     - Fixed real bugs: gets() (removed in modern C, unsafe) -> safeInput(),
       missing function prototypes causing implicit-declaration errors,
       Sleep() instead of sleep() (this is Windows, sleep() doesn't exist
       without unistd.h), fflush(stdin) replaced with a real input-buffer
       clear, scanf("%c") missing a leading space (was skipping/blocking
       on leftover newlines), undefined-behavior in getGroup/getRelationship
       when no case matched, PhoneNumber/Phone arrays too small to hold
       an 11-digit number + null terminator.

   NOTE: because a field (IsFavorite) was added to struct Contact, this
   version is NOT binary-compatible with an old contact.txt/id.txt from
   the 2022 program. Delete any old contact.txt / id.txt before running
   this version for the first time, so a fresh file is created.
   ======================================================================= */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <ctype.h>
#include <windows.h>
#include <conio.h>

/* ---------------------------- enums / types --------------------------- */

enum Type { Mobile = 1, Home = 2, Work = 3, Main = 4, Other = 5 };
enum Group { Emergency = 1, Colleague = 2, Family = 3, Friend = 4, Not_Available = 5 };
enum Relationship { Parent = 1, Mother = 2, Father = 3, Brother = 4, Sister = 5, Friends = 6, Relative = 7, Unknown = 8 };
typedef enum { False, True } boolean;

/* Windows console colors */
#define CLR_TITLE      11   /* bright cyan   */
#define CLR_MENU       14   /* yellow        */
#define CLR_TEXT       7    /* light gray    */
#define CLR_SUCCESS    10   /* bright green  */
#define CLR_ERROR      12   /* bright red    */
#define CLR_ACCENT     13   /* magenta       */
#define CLR_FAV        6    /* gold/brown    */

struct Contact {
    int Id;
    char CreateTime[50];
    char Name[100];
    char PhoneNumber[20];
    char Email[100];
    int Group;
    char Company[100];
    char Address[100];
    int Relationship;
    char Notes[100];
    char Phone[20];
    boolean IsBlocked;
    boolean IsFavorite;
};

struct Contact contacts[1000];
struct Contact contactList[1000];
int Total = 0;
int K = 0;
boolean showBlockContacts = False;
boolean showFavOnly = False;

/* ------------------------------ prototypes ----------------------------- */

void showTitle();
void showOptions();
void processUserOption();
void Reset();
void showContactActionTitle(const char a[100]);

void setColor(int color);
void resetColor();
void typewriter(const char *text, int delayMs, int color);
void loadingAnimation(const char *label, int steps);
void printDivider();
void safeInput(char *buffer, int size);
void clearInputBuffer();
void pauseKey();

int getId();
void setContactId(int currentId);
int checkDomain(const char *email, const char *domain);

struct Contact getContactInfo();
void sortContactList();
void AddToContactList(struct Contact contact);
void CreateNewContact();

const char *getGroup(int g);
const char *getRelationship(int r);
void showContacts(int i, struct Contact contact);

void loadContacts(int hide);
void ViewContactList();
void ViewContacts();

void blockAnyContact();
void BlockContact();
void unblockAnyContact();
void ViewBlockContact();

void RemoveAnyContact();
void RemoveContact();

struct Contact EditContactInfo(struct Contact contact);
void EditAnyContact();
void EditContact();

void findBy(int id, char *key);
void FindAnyContact();
void FindContact();

void toggleFavoriteAny();
void ToggleFavorite();
void showFavoriteList();
void ViewFavorites();

void showStatistics();
void exportContacts();

/* ------------------------------ UI helpers ------------------------------ */

void setColor(int color) {
    SetConsoleTextAttribute(GetStdHandle(STD_OUTPUT_HANDLE), color);
}

void resetColor() {
    setColor(CLR_TEXT);
}

void clearInputBuffer() {
    int c;
    while ((c = getchar()) != '\n' && c != EOF) { }
}

/* Reads a line safely (replaces the old unsafe gets()) and strips the newline */
void safeInput(char *buffer, int size) {
    if (fgets(buffer, size, stdin) != NULL) {
        size_t len = strlen(buffer);
        if (len > 0 && buffer[len - 1] == '\n') {
            buffer[len - 1] = '\0';
        } else {
            clearInputBuffer();
        }
    } else {
        buffer[0] = '\0';
    }
}

void typewriter(const char *text, int delayMs, int color) {
    setColor(color);
    for (int i = 0; text[i] != '\0'; i++) {
        putchar(text[i]);
        fflush(stdout);
        Sleep(delayMs);
    }
    resetColor();
}

void loadingAnimation(const char *label, int steps) {
    const char spinner[4] = { '|', '/', '-', '\\' };
    setColor(CLR_ACCENT);
    printf("\t\t\t%s ", label);
    for (int i = 0; i < steps; i++) {
        printf("%c", spinner[i % 4]);
        fflush(stdout);
        Sleep(120);
        printf("\b");
    }
    printf("[");
    setColor(CLR_SUCCESS);
    printf("done");
    setColor(CLR_ACCENT);
    printf("]\n");
    resetColor();
}

void printDivider() {
    setColor(CLR_ACCENT);
    printf("\t\t\t");
    for (int i = 0; i < 43; i++) printf("-");
    printf("\n");
    resetColor();
}

void pauseKey() {
    setColor(CLR_TEXT);
    printf("\n\t\t\tPress any key to show options....");
    getch();
}

/* --------------------------------- Menus --------------------------------- */

void showTitle() {
    printf("\n\n");
    setColor(CLR_TITLE);
    printf("\t\t\t=================================================\n");
    resetColor();
    typewriter("\t\t\t      DIGITAL PHONEBOOK -- 2026 EDITION\n", 8, CLR_TITLE);
    setColor(CLR_TITLE);
    printf("\t\t\t=================================================\n\n");
    resetColor();
}

void showOptions() {
    setColor(CLR_MENU);
    printf("\t\t\t 1. Create new contact\n");
    printf("\t\t\t 2. View all contacts\n");
    printf("\t\t\t 3. Edit contact\n");
    printf("\t\t\t 4. Find contact\n");
    printf("\t\t\t 5. Remove contact\n");
    printf("\t\t\t 6. Block contact\n");
    printf("\t\t\t 7. View blocked contacts\n");
    printf("\t\t\t 8. Mark / unmark favorite\n");
    printf("\t\t\t 9. View favorite contacts\n");
    printf("\t\t\t10. Contact statistics\n");
    printf("\t\t\t11. Export contacts to file\n");
    printf("\t\t\t12. Exit\n\n");
    resetColor();
    printf("\t\t\tSelect your option: ");
}

void showContactActionTitle(const char a[100]) {
    setColor(CLR_ACCENT);
    printf("\t\t\t%s\n", a);
    resetColor();
    printDivider();
}

void processUserOption() {
    int n;
    if (scanf("%d", &n) != 1) {
        clearInputBuffer();
        n = -1;
    }
    clearInputBuffer();

    switch (n) {
        case 1: CreateNewContact(); break;
        case 2: K = 0; showBlockContacts = False; ViewContacts(); break;
        case 3: K = 0; showBlockContacts = False; EditContact(); break;
        case 4: K = 0; showBlockContacts = False; FindContact(); break;
        case 5: K = 0; showBlockContacts = False; RemoveContact(); break;
        case 6: K = 0; showBlockContacts = False; BlockContact(); break;
        case 7: K = 0; showBlockContacts = True; ViewBlockContact(); break;
        case 8: K = 0; showBlockContacts = False; ToggleFavorite(); break;
        case 9: K = 0; showFavOnly = True; ViewFavorites(); break;
        case 10: showStatistics(); break;
        case 11: exportContacts(); break;
        case 12:
            system("cls");
            typewriter("\n\n\t\t\tThanks for using Digital Phonebook 2026. Goodbye!\n\n", 12, CLR_SUCCESS);
            exit(0);
            break;
        default:
            system("cls");
            showTitle();
            showOptions();
            setColor(CLR_ERROR);
            printf("\n\t\t\tInvalid option, please select again: ");
            resetColor();
            processUserOption();
            break;
    }
}

void Reset() {
    system("cls");
    showTitle();
    showOptions();
    processUserOption();
}

int main() {
    system("cls");
    showTitle();
    loadingAnimation("Initializing phonebook", 12);
    showOptions();
    processUserOption();
    return 0;
}

/* ------------------------------ ID handling ------------------------------ */

int getId() {
    FILE *fp;
    int id = 0;
    fp = fopen("id.txt", "r");
    if (fp != NULL) {
        fscanf(fp, "%d", &id);
        fclose(fp);
    }
    id = (id == 0) ? 1 : id + 1;

    fp = fopen("id.txt", "w");
    if (fp != NULL) {
        fprintf(fp, "%d", id);
        fclose(fp);
    }
    return id;
}

void setContactId(int currentId) {
    FILE *fp;
    if (currentId == 0) currentId = 1;
    fp = fopen("id.txt", "w");
    if (fp != NULL) {
        fprintf(fp, "%d", currentId);
        fclose(fp);
    }
}

int checkDomain(const char *email, const char *domain) {
    int slen = strlen(email);
    int domain_len = strlen(domain);
    if (domain_len > slen) return 0;
    if (strncmp(email + slen - domain_len, domain, domain_len) == 0) return 1;
    return 0;
}

/* --------------------------- Getting contact info -------------------------- */

struct Contact getContactInfo() {
    struct Contact contact;
    int valid = 0;
    int g, r;

    contact.Id = getId();

    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    strftime(contact.CreateTime, sizeof(contact.CreateTime), "%Y-%m-%d %H:%M", t);

    setColor(CLR_TEXT);
    printf("\t\t\tName: ");
    safeInput(contact.Name, sizeof(contact.Name));

    printf("\t\t\tEmail: ");
    safeInput(contact.Email, sizeof(contact.Email));
    valid = checkDomain(contact.Email, "@gmail.com") || checkDomain(contact.Email, "@yahoo.com") || checkDomain(contact.Email, "@outlook.com");
    while (!valid) {
        setColor(CLR_ERROR);
        printf("\t\t\tEmail must end with @gmail.com, @yahoo.com or @outlook.com\n");
        resetColor();
        printf("\t\t\tEmail: ");
        safeInput(contact.Email, sizeof(contact.Email));
        valid = checkDomain(contact.Email, "@gmail.com") || checkDomain(contact.Email, "@yahoo.com") || checkDomain(contact.Email, "@outlook.com");
    }

    printf("\t\t\tGroup (1.Emergency 2.Colleague 3.Family 4.Friend 5.Not Available): ");
    if (scanf("%d", &g) != 1) g = 5;
    clearInputBuffer();
    if (g < 1 || g > 5) g = 5;
    contact.Group = g;

    printf("\t\t\tCompany: ");
    safeInput(contact.Company, sizeof(contact.Company));

    printf("\t\t\tAddress: ");
    safeInput(contact.Address, sizeof(contact.Address));

    printf("\t\t\tRelationship (1.Parent 2.Mother 3.Father 4.Brother 5.Sister 6.Friends 7.Relative 8.Unknown): ");
    if (scanf("%d", &r) != 1) r = 8;
    clearInputBuffer();
    if (r < 1 || r > 8) r = 8;
    contact.Relationship = r;

    printf("\t\t\tNotes: ");
    safeInput(contact.Notes, sizeof(contact.Notes));

    printf("\t\t\tPhone: ");
    safeInput(contact.Phone, sizeof(contact.Phone));
    while (strlen(contact.Phone) > 15 || strlen(contact.Phone) < 10) {
        setColor(CLR_ERROR);
        printf("\t\t\tPhone number must be 10-15 digits.\n");
        resetColor();
        printf("\t\t\tPhone: ");
        safeInput(contact.Phone, sizeof(contact.Phone));
    }
    strncpy(contact.PhoneNumber, contact.Phone, sizeof(contact.PhoneNumber) - 1);
    contact.PhoneNumber[sizeof(contact.PhoneNumber) - 1] = '\0';

    contact.IsBlocked = False;
    contact.IsFavorite = False;
    resetColor();
    return contact;
}

void sortContactList() {
    int i, j;
    struct Contact temp;
    for (i = 0; i < Total; i++) {
        for (j = i + 1; j < Total; j++) {
            if (strcmp(contactList[i].Name, contactList[j].Name) > 0) {
                temp = contactList[i];
                contactList[i] = contactList[j];
                contactList[j] = temp;
            }
        }
    }
}

void AddToContactList(struct Contact contact) {
    FILE *fp;
    char ch;

    loadContacts(1);

    contactList[Total] = contact;
    Total = Total + 1;
    sortContactList();

    loadingAnimation("Saving new contact", 14);

    fp = fopen("contact.txt", "wb");
    fwrite(contactList, sizeof(struct Contact), Total, fp);
    fclose(fp);

    setColor(CLR_SUCCESS);
    printf("\t\t\tContact added successfully!\n");
    resetColor();
    printf("\t\t\tAdd another contact? (Y = Yes, N = No): ");
    scanf(" %c", &ch);
    clearInputBuffer();

    if (ch == 'Y' || ch == 'y') {
        CreateNewContact();
    } else {
        Reset();
    }
}

void CreateNewContact() {
    system("cls");
    showTitle();
    showContactActionTitle("Create Contact");
    struct Contact contact = getContactInfo();
    AddToContactList(contact);
}

/* ------------------------------ Lookups ------------------------------ */

const char *getGroup(int g) {
    switch (g) {
        case 1: return "Emergency";
        case 2: return "Colleague";
        case 3: return "Family";
        case 4: return "Friend";
        case 5: return "Not Available";
        default: return "Unknown";
    }
}

const char *getRelationship(int r) {
    switch (r) {
        case 1: return "Parent";
        case 2: return "Mother";
        case 3: return "Father";
        case 4: return "Brother";
        case 5: return "Sister";
        case 6: return "Friends";
        case 7: return "Relative";
        case 8: return "Unknown";
        default: return "Unknown";
    }
}

void showContacts(int i, struct Contact contact) {
    setColor(contact.IsFavorite ? CLR_FAV : CLR_ACCENT);
    printf("\t\t[%d]%s %s\n", i, contact.IsFavorite ? " *" : "", contact.Name);
    setColor(CLR_TEXT);
    printf("\t\t\tPhone: %s\n", contact.Phone);
    printf("\t\t\tEmail: %s\n", contact.Email);
    printf("\t\t\tGroup: %s\n", getGroup(contact.Group));
    printf("\t\t\tCompany: %s\n", contact.Company);
    printf("\t\t\tAddress: %s\n", contact.Address);
    printf("\t\t\tRelationship: %s\n", getRelationship(contact.Relationship));
    printf("\t\t\tNotes: %s\n", contact.Notes);
    printDivider();
    resetColor();
}

void loadContacts(int hide) {
    struct Contact contact;
    FILE *fp;
    int i = 0;

    Total = 0;
    K = 0;
    fp = fopen("contact.txt", "rb");
    if (fp == NULL) {
        if (hide == 0) {
            setColor(CLR_ERROR);
            printf("\t\t\tNo contacts found!\n");
            resetColor();
        }
        return;
    }

    if (hide == 0) loadingAnimation("Loading contacts", 10);

    while (fread(&contact, sizeof(contact), 1, fp) == 1) {
        contactList[i++] = contact;

        boolean matches;
        if (showFavOnly) {
            matches = contact.IsFavorite;
        } else {
            matches = (contact.IsBlocked == showBlockContacts);
        }

        if (matches) {
            K++;
            contacts[K - 1] = contact;
            if (hide == 0) showContacts(K, contact);
        }
    }
    Total = i;

    if (K < 1 && hide == 0) {
        setColor(CLR_ERROR);
        printf("\t\t\tNo contact found!\n");
        resetColor();
    }
    fclose(fp);
}

/* --------------------------------- View --------------------------------- */

void ViewContactList() {
    loadContacts(0);
    pauseKey();
    showFavOnly = False;
    Reset();
}

void ViewContacts() {
    system("cls");
    showTitle();
    showContactActionTitle("Your Contacts");
    ViewContactList();
}

/* -------------------------------- Block ---------------------------------- */

void blockAnyContact() {
    FILE *fp;
    int id, i, n;
    char ch;
    n = K;

    if (K == 0) { pauseKey(); Reset(); }

    printf("\n\t\t\tEnter contact Id to block (0 = Show Options): ");
    if (scanf("%d", &id) != 1) id = 0;
    clearInputBuffer();
    if (id == 0) Reset();

    if (id > K || id < 1) {
        setColor(CLR_ERROR);
        printf("\n\t\t\tNo contact found for Id : %d\n", id);
        resetColor();
        blockAnyContact();
    } else {
        for (i = 0; i < Total; i++) {
            if (contactList[i].Id == contacts[id - 1].Id) {
                contactList[i].IsBlocked = True;
                n = n - 1;
                break;
            }
        }
        fp = fopen("contact.txt", "wb");
        fwrite(contactList, sizeof(struct Contact), Total, fp);
        fclose(fp);
        setColor(CLR_SUCCESS);
        printf("\n\t\t\tContact blocked successfully!\n");
        resetColor();
    }

    if (n > 0) {
        printf("\n\t\t\tBlock another contact? (Y = Yes, N = No): ");
        ch = getch();
        if (ch == 'Y' || ch == 'y') { BlockContact(); }
    }
    pauseKey();
    Reset();
}

void BlockContact() {
    system("cls");
    showTitle();
    showContactActionTitle("Block Contact");
    loadContacts(0);
    blockAnyContact();
}

void unblockAnyContact() {
    FILE *fp;
    int id, i, n;
    char ch;
    n = K;

    if (K == 0) { pauseKey(); Reset(); }

    printf("\n\t\t\tEnter contact Id to unblock (0 = Show Options): ");
    if (scanf("%d", &id) != 1) id = 0;
    clearInputBuffer();
    if (id == 0) Reset();

    if (id > K || id < 1) {
        setColor(CLR_ERROR);
        printf("\n\t\t\tNo contact found for Id : %d\n", id);
        resetColor();
    } else {
        for (i = 0; i < Total; i++) {
            if (contactList[i].Id == contacts[id - 1].Id) {
                contactList[i].IsBlocked = False;
                n = n - 1;
                break;
            }
        }
        fp = fopen("contact.txt", "wb");
        fwrite(contactList, sizeof(struct Contact), Total, fp);
        fclose(fp);
        setColor(CLR_SUCCESS);
        printf("\n\t\t\tContact unblocked successfully!\n");
        resetColor();
    }

    if (n > 0) {
        printf("\n\t\t\tUnblock another contact? (Y = Yes, N = No): ");
        ch = getch();
        if (ch == 'Y' || ch == 'y') { ViewBlockContact(); }
    }
    pauseKey();
    Reset();
}

void ViewBlockContact() {
    system("cls");
    showTitle();
    showContactActionTitle("Blocked Contacts");
    loadContacts(0);
    unblockAnyContact();
}

/* -------------------------------- Remove --------------------------------- */

void RemoveAnyContact() {
    FILE *fp;
    int id, i, n;
    char ch;
    n = K;

    if (K == 0) { pauseKey(); Reset(); }

    printf("\n\t\t\tEnter contact Id to remove (0 = Show Options): ");
    if (scanf("%d", &id) != 1) id = 0;
    clearInputBuffer();
    if (id == 0) Reset();

    if (id > K || id < 1) {
        setColor(CLR_ERROR);
        printf("\n\t\t\tNo contact found for Id : %d\n", id);
        resetColor();
        RemoveAnyContact();
    } else {
        int targetId = contacts[id - 1].Id;
        int pos = -1;
        for (i = 0; i < Total; i++) {
            if (contactList[i].Id == targetId) { pos = i; break; }
        }
        if (pos != -1) {
            for (i = pos; i < Total - 1; i++) {
                contactList[i] = contactList[i + 1];
            }
            Total = Total - 1;
            fp = fopen("contact.txt", "wb");
            fwrite(contactList, sizeof(struct Contact), Total, fp);
            fclose(fp);
            setColor(CLR_SUCCESS);
            printf("\n\t\t\tContact removed successfully!\n");
            resetColor();
        }
    }

    if (n > 0) {
        printf("\n\t\t\tRemove another contact? (Y = Yes, N = No): ");
        ch = getch();
        if (ch == 'Y' || ch == 'y') { RemoveContact(); }
    }
    pauseKey();
    Reset();
}

void RemoveContact() {
    system("cls");
    showTitle();
    showContactActionTitle("Remove Contact");
    loadContacts(0);
    RemoveAnyContact();
}

/* --------------------------------- Edit ----------------------------------- */

struct Contact EditContactInfo(struct Contact contact) {
    char ch;
    int g, r;
    char buf[100];

    printf("\n\t\t\tUpdate Name? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Name: ", contact.Name);
        safeInput(contact.Name, sizeof(contact.Name));
    }

    printf("\n\t\t\tUpdate Phone? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Phone: ", contact.Phone);
        safeInput(buf, sizeof(buf));
        strncpy(contact.Phone, buf, sizeof(contact.Phone) - 1);
        contact.Phone[sizeof(contact.Phone) - 1] = '\0';
        strncpy(contact.PhoneNumber, buf, sizeof(contact.PhoneNumber) - 1);
        contact.PhoneNumber[sizeof(contact.PhoneNumber) - 1] = '\0';
    }

    printf("\n\t\t\tUpdate Email? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Email: ", contact.Email);
        safeInput(contact.Email, sizeof(contact.Email));
    }

    printf("\n\t\t\tUpdate Group? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Group (1-5): ", getGroup(contact.Group));
        if (scanf("%d", &g) != 1) g = contact.Group;
        clearInputBuffer();
        if (g >= 1 && g <= 5) contact.Group = g;
    }

    printf("\n\t\t\tUpdate Company? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Company: ", contact.Company);
        safeInput(contact.Company, sizeof(contact.Company));
    }

    printf("\n\t\t\tUpdate Address? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Address: ", contact.Address);
        safeInput(contact.Address, sizeof(contact.Address));
    }

    printf("\n\t\t\tUpdate Relationship? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Relationship (1-8): ", getRelationship(contact.Relationship));
        if (scanf("%d", &r) != 1) r = contact.Relationship;
        clearInputBuffer();
        if (r >= 1 && r <= 8) contact.Relationship = r;
    }

    printf("\n\t\t\tUpdate Notes? (Y/N): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') {
        clearInputBuffer();
        printf("\n\t\t\tCurrent: %s\n\t\t\tNew Notes: ", contact.Notes);
        safeInput(contact.Notes, sizeof(contact.Notes));
    }

    return contact;
}

void EditAnyContact() {
    FILE *fp;
    int id, i, n;
    char ch;
    struct Contact contact;
    n = K;

    if (K == 0) { pauseKey(); Reset(); }

    printf("\n\t\t\tEnter contact Id to edit (0 = Show Options): ");
    if (scanf("%d", &id) != 1) id = 0;
    clearInputBuffer();
    if (id == 0) Reset();

    if (id > K || id < 1) {
        setColor(CLR_ERROR);
        printf("\n\t\t\tNo contact found for Id : %d\n", id);
        resetColor();
        EditAnyContact();
    } else {
        for (i = 0; i < Total; i++) {
            if (contactList[i].Id == contacts[id - 1].Id) contact = contactList[i];
        }
        contact = EditContactInfo(contact);
        for (i = 0; i < Total; i++) {
            if (contactList[i].Id == contact.Id) contactList[i] = contact;
        }
        sortContactList();
        fp = fopen("contact.txt", "wb");
        fwrite(contactList, sizeof(struct Contact), Total, fp);
        fclose(fp);
        setColor(CLR_SUCCESS);
        printf("\n\n\t\t\tContact edited successfully!\n");
        resetColor();
    }

    if (n > 0) {
        printf("\n\t\t\tEdit another contact? (Y = Yes, N = No): ");
        ch = getch();
        if (ch == 'Y' || ch == 'y') { EditContact(); }
    }
    pauseKey();
    Reset();
}

void EditContact() {
    system("cls");
    showTitle();
    showContactActionTitle("Edit Contact");
    loadContacts(0);
    EditAnyContact();
}

/* --------------------------------- Find ----------------------------------- */

void findBy(int id, char *key) {
    struct Contact result[1000];
    int i, k = 0;
    for (i = 0; i < K; i++) {
        int hit = 0;
        if (id == 1 && strstr(contacts[i].Name, key)) hit = 1;
        else if (id == 2 && strstr(contacts[i].Phone, key)) hit = 1;
        else if (id == 3 && strstr(contacts[i].Email, key)) hit = 1;
        else if (id == 4 && strstr(contacts[i].Company, key)) hit = 1;
        else if (id == 5 && strstr(contacts[i].Address, key)) hit = 1;
        else if (id == 6 && strstr(contacts[i].Notes, key)) hit = 1;

        if (hit) result[k++] = contacts[i];
    }

    if (k == 0) {
        setColor(CLR_ERROR);
        printf("\n\t\t\tNo contact found for search key : %s\n", key);
        resetColor();
    } else {
        for (i = 0; i < k; i++) showContacts(i + 1, result[i]);
    }
}

void FindAnyContact() {
    int id;
    char ch;
    char key[100];

    if (Total == 0) { pauseKey(); Reset(); }

    printf("\n\t\t\tFind by (1.Name 2.Phone 3.Email 4.Company 5.Address 6.Notes) (0 = Show Options): ");
    if (scanf("%d", &id) != 1) id = 0;
    clearInputBuffer();

    if (id == 0) {
        Reset();
    } else {
        printf("\n\t\t\tEnter search key: ");
        safeInput(key, sizeof(key));
        if (id > 6 || id < 1) id = 1;
        findBy(id, key);
    }

    printf("\n\t\t\tFind another contact? (Y = Yes, N = No): ");
    ch = getch();
    if (ch == 'Y' || ch == 'y') { FindContact(); }
    pauseKey();
    Reset();
}

void FindContact() {
    system("cls");
    showTitle();
    showContactActionTitle("Find Contact");
    loadContacts(1);
    FindAnyContact();
}

/* ------------------------------- Favorites --------------------------------- */

void toggleFavoriteAny() {
    FILE *fp;
    int id, i;

    if (K == 0) { pauseKey(); Reset(); }

    printf("\n\t\t\tEnter contact Id to mark/unmark favorite (0 = Show Options): ");
    if (scanf("%d", &id) != 1) id = 0;
    clearInputBuffer();
    if (id == 0) Reset();

    if (id > K || id < 1) {
        setColor(CLR_ERROR);
        printf("\n\t\t\tNo contact found for Id : %d\n", id);
        resetColor();
    } else {
        for (i = 0; i < Total; i++) {
            if (contactList[i].Id == contacts[id - 1].Id) {
                contactList[i].IsFavorite = (contactList[i].IsFavorite == True) ? False : True;
                setColor(CLR_SUCCESS);
                printf("\n\t\t\t%s %s favorites!\n", contactList[i].Name,
                       contactList[i].IsFavorite == True ? "added to" : "removed from");
                resetColor();
                break;
            }
        }
        fp = fopen("contact.txt", "wb");
        fwrite(contactList, sizeof(struct Contact), Total, fp);
        fclose(fp);
    }
    pauseKey();
    Reset();
}

void ToggleFavorite() {
    system("cls");
    showTitle();
    showContactActionTitle("Mark / Unmark Favorite");
    showFavOnly = False;
    loadContacts(0);
    toggleFavoriteAny();
}

void ViewFavorites() {
    system("cls");
    showTitle();
    showContactActionTitle("Favorite Contacts");
    loadContacts(0);
    showFavOnly = False;
    pauseKey();
    Reset();
}

/* ------------------------------- Statistics --------------------------------- */

void showStatistics() {
    FILE *fp;
    struct Contact c;
    int total = 0, blocked = 0, favorites = 0;
    int groupCount[6] = {0};
    int relCount[9] = {0};

    system("cls");
    showTitle();
    showContactActionTitle("Contact Statistics");

    fp = fopen("contact.txt", "rb");
    if (fp != NULL) {
        while (fread(&c, sizeof(c), 1, fp) == 1) {
            total++;
            if (c.IsBlocked == True) blocked++;
            if (c.IsFavorite == True) favorites++;
            if (c.Group >= 1 && c.Group <= 5) groupCount[c.Group]++;
            if (c.Relationship >= 1 && c.Relationship <= 8) relCount[c.Relationship]++;
        }
        fclose(fp);
    }

    loadingAnimation("Crunching numbers", 10);

    setColor(CLR_TEXT);
    printf("\t\t\tTotal contacts   : %d\n", total);
    setColor(CLR_ERROR);
    printf("\t\t\tBlocked contacts : %d\n", blocked);
    setColor(CLR_FAV);
    printf("\t\t\tFavorite contacts: %d\n", favorites);
    resetColor();
    printDivider();

    setColor(CLR_MENU);
    printf("\t\t\tBy Group:\n");
    resetColor();
    for (int g = 1; g <= 5; g++) {
        printf("\t\t\t  %-14s: %d\n", getGroup(g), groupCount[g]);
    }
    printDivider();

    setColor(CLR_MENU);
    printf("\t\t\tBy Relationship:\n");
    resetColor();
    for (int r = 1; r <= 8; r++) {
        printf("\t\t\t  %-10s: %d\n", getRelationship(r), relCount[r]);
    }

    pauseKey();
    Reset();
}

/* -------------------------------- Export ------------------------------------ */

void exportContacts() {
    FILE *in, *out;
    struct Contact c;
    int count = 0;

    system("cls");
    showTitle();
    showContactActionTitle("Export Contacts");

    in = fopen("contact.txt", "rb");
    if (in == NULL) {
        setColor(CLR_ERROR);
        printf("\t\t\tNo contacts to export!\n");
        resetColor();
        pauseKey();
        Reset();
    }

    out = fopen("contacts_export.txt", "w");
    fprintf(out, "DIGITAL PHONEBOOK 2026 -- EXPORTED CONTACT LIST\n");
    fprintf(out, "=================================================\n\n");

    while (fread(&c, sizeof(c), 1, in) == 1) {
        count++;
        fprintf(out, "%d. %s%s\n", count, c.Name, c.IsFavorite == True ? "  [Favorite]" : "");
        fprintf(out, "   Phone       : %s\n", c.Phone);
        fprintf(out, "   Email       : %s\n", c.Email);
        fprintf(out, "   Group       : %s\n", getGroup(c.Group));
        fprintf(out, "   Company     : %s\n", c.Company);
        fprintf(out, "   Address     : %s\n", c.Address);
        fprintf(out, "   Relationship: %s\n", getRelationship(c.Relationship));
        fprintf(out, "   Notes       : %s\n", c.Notes);
        fprintf(out, "   Status      : %s\n\n", c.IsBlocked == True ? "Blocked" : "Active");
    }
    fclose(in);
    fclose(out);

    loadingAnimation("Writing contacts_export.txt", 14);

    setColor(CLR_SUCCESS);
    printf("\t\t\tExported %d contact(s) to contacts_export.txt\n", count);
    resetColor();

    pauseKey();
    Reset();
}
