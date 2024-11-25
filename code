#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdarg.h>
#include <stdint.h>
#include <sys/mman.h>
#include <unistd.h>

#pragma clang assume_nonnull begin

static long PAGE_SIZE = 0;
static void *apply(void *fn, void *arg)
{
    if (PAGE_SIZE == 0)
        PAGE_SIZE = sysconf(_SC_PAGESIZE);
    if (PAGE_SIZE == -1) {
        perror("sysconf");
        exit(EXIT_FAILURE);
    }

    uint8_t *code = mmap(nullptr, PAGE_SIZE, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (code == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    uint64_t offset = 0;

    // We need 16 byte alignment for the call instruction \\
    // SUB RSP, 8
    code[offset++] = 0x48;
    code[offset++] = 0x83;
    code[offset++] = 0xEC;
    code[offset++] = 0x08;

    // MOV RDI, IMM64 (arg)
    code[offset++] = 0x48;
    code[offset++] = 0xBF;
    memcpy(&code[offset], &arg, sizeof(uintptr_t));
    offset += sizeof(uintptr_t);

    // MOV RAX, IMM64 (&fn)
    code[offset++] = 0x48;
    code[offset++] = 0xB8;
    memcpy(&code[offset], &fn, sizeof(uintptr_t));
    offset += sizeof(uintptr_t);

    // CALL RAX
    code[offset++] = 0xFF;
    code[offset++] = 0xD0;

    // ADD RSP, 8
    code[offset++] = 0x48;
    code[offset++] = 0x83;
    code[offset++] = 0xC4;
    code[offset++] = 0x08;

    // RET
    code[offset++] = 0xC3;

    if (mprotect(code, PAGE_SIZE, PROT_READ | PROT_EXEC) == -1) {
        perror("mprotect");
        exit(EXIT_FAILURE);
    }

    return code;
}

static void free_method(void *fn)
{
    munmap(fn, PAGE_SIZE);
}

struct Person {
    const char *name;
    int age;

    const char *(*tostring)();
    void (*destroy)();
};

static const char *Person_tostring(struct Person *self)
{
    char *buf = malloc(256);
    snprintf(buf, 256, "Person(%s, %d)", self->name, self->age);
    return buf;
}

static void Person_destroy(struct Person *self)
{
    free((void *)self->name);
    free_method(self->tostring);
}

static void create_person(struct Person *self, const char *name, int age)
{
    self->name = strdup(name);
    self->age = age;
    self->tostring = apply(Person_tostring, self);
    self->destroy = apply(Person_destroy, self);
}


int main(int argc, const char *_Nonnull argv[static _Nonnull argc])
{
    struct Person person;
    create_person(&person, "John Doe", 30);
    printf("Getting person\n");
    printf("%s\n", person.tostring());
    person.destroy();
}

#pragma clang assume_nonnull end
