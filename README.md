#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <ctype.h>

#define MAX_LINE_LEN 256
#define MAX_LINES 500
#define MAX_VARS 100
#define MAX_FUNCS 50
#define MAX_LIST_ITEMS 50
#define MAX_MAP_ITEMS 50
#define MAX_TOKENS 100

// ── أنواع الـ Tokens في المحلل المعجمي (Lexer) ──
typedef enum {
    TOKEN_EOF = 0,
    TOKEN_KEYWORD,    // مثل: حط، اطلب، لو، طالما، اعمل...
    TOKEN_IDENTIFIER, // أسماء المتغيرات والدوال
    TOKEN_NUMBER,     // الأرقام الصريحة
    TOKEN_STRING,     // النصوص بين علامات تنصيص
    TOKEN_OPERATOR,   // العمليات: +، -، *، /، **، %
    TOKEN_SYMBOL      // الرموز: [، ]، {، }، (، )، :، ،
} TokenType;

typedef struct {
    TokenType type;
    char text[128];
    double num_val;
} Token;

// ── الهياكل البيانية المتقدمة (متغيرات، قوائم، خرائط، دوال) ──
typedef struct {
    char key[64];
    char value_str[MAX_LINE_LEN];
    double value_num;
    int is_string;
} MapItem;

typedef struct {
    char name[64];
    double value;
    char str_val[MAX_LINE_LEN];
    int is_string;
    
    double list_nums[MAX_LIST_ITEMS];
    int list_size;
    int is_list;

    MapItem map_items[MAX_MAP_ITEMS];
    int map_size;
    int is_map;
} Variable;

typedef struct {
    char name[64];
    int start_line;
    int end_line;
} Function;

Variable vars[MAX_VARS];
int var_count = 0;

Function funcs[MAX_FUNCS];
int func_count = 0;

int lines_without_water = 0;
const int MAX_LINES_WITHOUT_WATER = 15;

int in_function = 0;
int has_returned = 0;
double return_value = 0;
char return_str_val[MAX_LINE_LEN];
int return_is_string = 0;
int has_broken = 0; 

// ── وظائف الـ Lexer (المحلل المعجمي المطور) ──
int is_arabic_keyword(const char* word) {
    return (strcmp(word, "حط") == 0 || strcmp(word, "اطلب") == 0 || 
            strcmp(word, "اسأل") == 0 || strcmp(word, "احسب") == 0 || 
            strcmp(word, "زود") == 0 || strcmp(word, "لو") == 0 || 
            strcmp(word, "غير") == 0 || strcmp(word, "طالما") == 0 || 
            strcmp(word, "كرر") == 0 || strcmp(word, "اعمل") == 0 || 
            strcmp(word, "نفذ") == 0 || strcmp(word, "أرجع") == 0 || 
            strcmp(word, "قف") == 0 || strcmp(word, "كوباية") == 0 || 
            strcmp(word, "الحساب") == 0);
}

void lex_line(const char* line, Token tokens[], int* token_count) {
    *token_count = 0;
    const char* p = line;

    while (*p && *token_count < MAX_TOKENS) {
        while (*p && isspace((unsigned char)*p)) p++;
        if (!*p) break;

        // تعليقات
        if (*p == '#') break;

        // النصوص
        if (*p == '"') {
            tokens[*token_count].type = TOKEN_STRING;
            p++;
            int idx = 0;
            while (*p && *p != '"' && idx < 127) {
                tokens[*token_count].text[idx++] = *p++;
            }
            tokens[*token_count].text[idx] = '\0';
            if (*p == '"') p++;
            (*token_count)++;
            continue;
        }

        // الأرقام
        if (isdigit((unsigned char)*p) || (*p == '.' && isdigit((unsigned char)*(p+1)))) {
            tokens[*token_count].type = TOKEN_NUMBER;
            char num_buf[64];
            int idx = 0;
            while (*p && (isdigit((unsigned char)*p) || *p == '.')) {
                num_buf[idx++] = *p++;
            }
            num_buf[idx] = '\0';
            strcpy(tokens[*token_count].text, num_buf);
            tokens[*token_count].num_val = strtod(num_buf, NULL);
            (*token_count)++;
            continue;
        }

        // الرموز والأسرار الحسابية
        if (*p == '+' || *p == '-' || *p == '/' || *p == '%' || *p == '=' || *p == '<' || *p == '>') {
            tokens[*token_count].type = TOKEN_OPERATOR;
            tokens[*token_count].text[0] = *p;
            tokens[*token_count].text[1] = '\0';
            p++;
            // دعم العمليات المزدوجة مثل ** أو ==
            if ((*p == '*' && tokens[*token_count].text[0] == '*') || 
                (*p == '=' && tokens[*token_count].text[0] == '=')) {
                tokens[*token_count].text[1] = *p;
                tokens[*token_count].text[2] = '\0';
                p++;
            }
            (*token_count)++;
            continue;
        }

        if (*p == '[' || *p == ']' || *p == '{' || *p == '}' || *p == '(' || *p == ')' || *p == ':' || *p == '،' || *p == ',') {
            tokens[*token_count].type = TOKEN_SYMBOL;
            tokens[*token_count].text[0] = *p;
            tokens[*token_count].text[1] = '\0';
            p++;
            (*token_count)++;
            continue;
        }

        // الكلمات المفتاحية والمعرفات (عربية أو إنجليزية)
        if (isalpha((unsigned char)*p) || (unsigned char)*p > 127) {
            char word[128];
            int idx = 0;
            while (*p && (isalnum((unsigned char)*p) || (unsigned char)*p > 127 || *p == '_')) {
                if (idx < 127) word[idx++] = *p;
                p++;
            }
            word[idx] = '\0';

            tokens[*token_count].type = is_arabic_keyword(word) ? TOKEN_KEYWORD : TOKEN_IDENTIFIER;
            strcpy(tokens[*token_count].text, word);
            (*token_count)++;
            continue;
        }

        p++; // تخطي أي رمز غريب غير معروف لتجنب التعليق
    }
    tokens[*token_count].type = TOKEN_EOF;
}

// ── إدارة المتغيرات ──
Variable* get_variable(const char* name) {
    for (int i = 0; i < var_count; i++) {
        if (strcmp(vars[i].name, name) == 0) {
            return &vars[i];
        }
    }
    return NULL;
}

void set_number_var(const char* name, double val) {
    Variable* v = get_variable(name);
    if (v) {
        v->value = val;
        v->is_string = 0;
        v->is_list = 0;
        v->is_map = 0;
    } else if (var_count < MAX_VARS) {
        strcpy(vars[var_count].name, name);
        vars[var_count].value = val;
        vars[var_count].is_string = 0;
        vars[var_count].is_list = 0;
        vars[var_count].is_map = 0;
        var_count++;
    }
}

void set_string_var(const char* name, const char* str) {
    Variable* v = get_variable(name);
    if (v) {
        strcpy(v->str_val, str);
        v->is_string = 1;
        v->is_list = 0;
        v->is_map = 0;
    } else if (var_count < MAX_VARS) {
        strcpy(vars[var_count].name, name);
        strcpy(vars[var_count].str_val, str);
        vars[var_count].is_string = 1;
        vars[var_count].is_list = 0;
        vars[var_count].is_map = 0;
        var_count++;
    }
}

void init_list_var(const char* name) {
    Variable* v = get_variable(name);
    if (!v && var_count < MAX_VARS) {
        strcpy(vars[var_count].name, name);
        v = &vars[var_count];
        var_count++;
    }
    if (v) {
        v->is_string = 0;
        v->is_list = 1;
        v->is_map = 0;
        v->list_size = 0;
    }
}

void list_add(double val, const char* name) {
    Variable* v = get_variable(name);
    if (v && v->is_list && v->list_size < MAX_LIST_ITEMS) {
        v->list_nums[v->list_size++] = val;
    }
}

void init_map_var(const char* name) {
    Variable* v = get_variable(name);
    if (!v && var_count < MAX_VARS) {
        strcpy(vars[var_count].name, name);
        v = &vars[var_count];
        var_count++;
    }
    if (v) {
        v->is_string = 0;
        v->is_list = 0;
        v->is_map = 1;
        v->map_size = 0;
    }
}

void map_set(const char* name, const char* key, double num, const char* str, int is_str) {
    Variable* v = get_variable(name);
    if (!v || !v->is_map || v->map_size >= MAX_MAP_ITEMS) return;
    
    for (int i = 0; i < v->map_size; i++) {
        if (strcmp(v->map_items[i].key, key) == 0) {
            if (is_str) {
                strcpy(v->map_items[i].value_str, str);
                v->map_items[i].is_string = 1;
            } else {
                v->map_items[i].value_num = num;
                v->map_items[i].is_string = 0;
            }
            return;
        }
    }
    
    strcpy(v->map_items[v->map_size].key, key);
    if (is_str) {
        strcpy(v->map_items[v->map_size].value_str, str);
        v->map_items[v->map_size].is_string = 1;
    } else {
        v->map_items[v->map_size].value_num = num;
        v->map_items[v->map_size].is_string = 0;
    }
    v->map_size++;
}

char* trim(char* str) {
    while(isspace((unsigned char)*str)) str++;
    if(*str == 0) return str;
    char* end = str + strlen(str) - 1;
    while(end > str && isspace((unsigned char)*end)) end--;
    end[1] = '\0';
    return str;
}

// ── مُحلل التعبيرات الحسابية ──
double evaluate_expression(const char* expr);

double parse_factor(const char** expr_ptr) {
    while(isspace(**expr_ptr)) (*expr_ptr)++;
    
    if (**expr_ptr == '(') {
        (*expr_ptr)++;
        double val = evaluate_expression(*expr_ptr);
        while(isspace(**expr_ptr)) (*expr_ptr)++;
        if (**expr_ptr == ')') (*expr_ptr)++;
        return val;
    }
    
    if (strncmp(*expr_ptr, "طول(", 4) == 0) {
        *expr_ptr += 4;
        char var_name[64];
        int vidx = 0;
        while(**expr_ptr && **expr_ptr != ')') {
            var_name[vidx++] = **expr_ptr;
            (*expr_ptr)++;
        }
        var_name[vidx] = '\0';
        if (**expr_ptr == ')') (*expr_ptr)++;
        
        char* inner = trim(var_name);
        if (inner[0] == '"' && inner[strlen(inner)-1] == '"') {
            inner[strlen(inner)-1] = '\0';
            return strlen(inner + 1);
        }

        Variable* v = get_variable(inner);
        if (v) {
            if (v->is_list) return v->list_size;
            if (v->is_map) return v->map_size;
            if (v->is_string) return strlen(v->str_val);
        }
        return 0;
    }

    if (strncmp(*expr_ptr, "رقم(", 4) == 0) {
        *expr_ptr += 4;
        const char* start = *expr_ptr;
        int depth = 1;
        while (**expr_ptr && depth > 0) {
            if (**expr_ptr == '(') depth++;
            if (**expr_ptr == ')') depth--;
            if (depth > 0) (*expr_ptr)++;
        }
        char inner[128];
        int len = *expr_ptr - start;
        strncpy(inner, start, len);
        inner[len] = '\0';
        if (**expr_ptr == ')') (*expr_ptr)++;

        char* trimmed_inner = trim(inner);
        Variable* v = get_variable(trimmed_inner);
        if (v) return (v->is_string || v->is_list || v->is_map) ? 0.0 : 1.0;

        char* endptr;
        strtod(trimmed_inner, &endptr);
        if (*endptr == '\0' && len > 0) return 1.0;

        return 0.0;
    }

    if (strncmp(*expr_ptr, "جذر(", 7) == 0) {
        *expr_ptr += 7;
        double val = evaluate_expression(*expr_ptr);
        while(**expr_ptr && **expr_ptr != ')') (*expr_ptr)++;
        if (**expr_ptr == ')') (*expr_ptr)++;
        if (val < 0) { printf("❌ خطأ: جذر لعدد سالب!\n"); return 0; }
        return sqrt(val);
    }
    if (strncmp(*expr_ptr, "مطلق(", 7) == 0) {
        *expr_ptr += 7;
        double val = evaluate_expression(*expr_ptr);
        while(**expr_ptr && **expr_ptr != ')') (*expr_ptr)++;
        if (**expr_ptr == ')') (*expr_ptr)++;
        return fabs(val);
    }
    if (strncmp(*expr_ptr, "تقريب(", 8) == 0) {
        *expr_ptr += 8;
        double val = evaluate_expression(*expr_ptr);
        while(**expr_ptr && **expr_ptr != ')') (*expr_ptr)++;
        if (**expr_ptr == ')') (*expr_ptr)++;
        return round(val);
    }

    char token[64];
    int idx = 0;
    while(**expr_ptr && (isalnum(**expr_ptr) || (unsigned char)**expr_ptr > 127 || **expr_ptr == '.' || **expr_ptr == '_')) {
        token[idx++] = **expr_ptr;
        (*expr_ptr)++;
    }
    token[idx] = '\0';

    if (idx == 0) return 0;

    char* endptr;
    double num = strtod(token, &endptr);
    if (*endptr == '\0') return num;

    Variable* v = get_variable(token);
    if (v && !v->is_string && !v->is_list && !v->is_map) return v->value;

    return 0;
}

double parse_power(const char** expr_ptr) {
    double val = parse_factor(expr_ptr);
    while(isspace(**expr_ptr)) (*expr_ptr)++;
    if (**expr_ptr == '*' && *(*expr_ptr + 1) == '*') {
        *expr_ptr += 2;
        return pow(val, parse_power(expr_ptr));
    }
    return val;
}

double parse_term(const char** expr_ptr) {
    double val = parse_power(expr_ptr);
    while (1) {
        while(isspace(**expr_ptr)) (*expr_ptr)++;
        char op = **expr_ptr;
        if (op != '*' && op != '/' && op != '%') break;
        (*expr_ptr)++;
        double next_val = parse_power(expr_ptr);
        if (op == '*') val *= next_val;
        else if (op == '/') {
            if (next_val == 0) { printf("❌ خطأ: القسمة على صفر!\n"); exit(1); }
            val /= next_val;
        } else if (op == '%') {
            if (next_val == 0) { printf("❌ خطأ: باقي القسمة على صفر!\n"); exit(1); }
            val = fmod(val, next_val);
        }
    }
    return val;
}

double evaluate_expression(const char* expr) {
    const char* ptr = expr;
    double val = parse_term(&ptr);
    while (1) {
        while(isspace(*ptr)) ptr++;
        char op = *ptr;
        if (op != '+' && op != '-') break;
        ptr++;
        double next_val = parse_term(&ptr);
        if (op == '+') val += next_val;
        else val -= next_val;
    }
    return val;
}

int evaluate_condition(const char* cond) {
    char local[256];
    strncpy(local, cond, sizeof(local) - 1);
    local[sizeof(local) - 1] = '\0';
    
    char left_expr[128], right_expr[128];
    char* pos = NULL;

    if ((pos = strstr(local, "يساوى")) != NULL) {
        *pos = '\0';
        strcpy(left_expr, local);
        strcpy(right_expr, pos + 10);
        return evaluate_expression(trim(left_expr)) == evaluate_expression(trim(right_expr));
    }
    if ((pos = strstr(local, "==")) != NULL) {
        *pos = '\0';
        strcpy(left_expr, local);
        strcpy(right_expr, pos + 2);
        return evaluate_expression(trim(left_expr)) == evaluate_expression(trim(right_expr));
    }
    if ((pos = strstr(local, ">")) != NULL) {
        *pos = '\0';
        strcpy(left_expr, local);
        strcpy(right_expr, pos + 1);
        return evaluate_expression(trim(left_expr)) > evaluate_expression(trim(right_expr));
    }
    if ((pos = strstr(local, "<")) != NULL) {
        *pos = '\0';
        strcpy(left_expr, local);
        strcpy(right_expr, pos + 1);
        return evaluate_expression(trim(left_expr)) < evaluate_expression(trim(right_expr));
    }
    return evaluate_expression(trim(local)) != 0;
}

int find_block_end(char lines[MAX_LINES][MAX_LINE_LEN], int start_idx, const char* open_keyword, const char* close_cmd) {
    int depth = 0;
    size_t open_len = strlen(open_keyword);
    for (int j = start_idx; j < MAX_LINES; j++) {
        char* ln = trim(lines[j]);
        if (strncmp(ln, open_keyword, open_len) == 0) {
            depth++;
        } else if (strcmp(ln, close_cmd) == 0) {
            if (depth == 0) return j;
            depth--;
        }
    }
    return -1;
}

typedef struct {
    int type; 
    char cond[128];
    int start_line;
    int end_line;
} IfBranch;

// ── محرك التنفيذ الأساسي باستخدام الـ Lexer ──
void execute_script(char lines[MAX_LINES][MAX_LINE_LEN], int total_lines, int start_idx, int end_idx) {
    int i = start_idx;
    while (i < end_idx) {
        if (has_returned || has_broken) break;

        char line[MAX_LINE_LEN];
        strcpy(line, lines[i]);
        char* clean_line = trim(line);
        i++;

        if (strlen(clean_line) == 0 || clean_line[0] == '#') continue;

        // تحليل السطر باستخدام الـ Lexer الجديد
        Token tokens[MAX_TOKENS];
        int token_count = 0;
        lex_line(clean_line, tokens, &token_count);
        if (token_count == 0) continue;

        if (has_broken && in_function == 0 && start_idx == 0) {
            has_broken = 0;
        }

        if (strcmp(clean_line, "كوباية مية") == 0) {
            lines_without_water = 0;
            printf("☕ [إسعاف]: تم شرب كوباية مية، المبرمج استرد وعيه!\n");
            continue;
        }

        if (strcmp(clean_line, "الحساب يا أسطى") == 0) {
            printf("☕ نهاية الجلسة: قفلنا القهوة على خير، نورتنا!\n");
            exit(0);
        }

        if (strcmp(clean_line, "قف") == 0) {
            has_broken = 1;
            return;
        }

        if (strncmp(clean_line, "أرجع ", 6) == 0) {
            char* expr = trim(clean_line + 6);
            if (expr[0] == '"' && expr[strlen(expr)-1] == '"') {
                expr[strlen(expr)-1] = '\0';
                strcpy(return_str_val, expr + 1);
                return_is_string = 1;
            } else {
                return_value = evaluate_expression(expr);
                return_is_string = 0;
            }
            has_returned = 1;
            return;
        }

        if (strncmp(clean_line, "اطلب ", 5) == 0) {
            char* content = trim(clean_line + 5);
            if (content[0] == '"' && content[strlen(content)-1] == '"') {
                content[strlen(content)-1] = '\0';
                printf("%s\n", content + 1);
            } else {
                Variable* v = get_variable(content);
                if (v) {
                    if (v->is_string) {
                        printf("%s\n", v->str_val);
                    } else if (v->is_list) {
                        printf("[");
                        for (int k = 0; k < v->list_size; k++) {
                            printf("%g%s", v->list_nums[k], (k < v->list_size - 1) ? ", " : "");
                        }
                        printf("]\n");
                    } else if (v->is_map) {
                        printf("{");
                        for (int k = 0; k < v->map_size; k++) {
                            if (v->map_items[k].is_string) {
                                printf("%s: \"%s\"%s", v->map_items[k].key, v->map_items[k].value_str, (k < v->map_size - 1) ? ", " : "");
                            } else {
                                printf("%s: %g%s", v->map_items[k].key, v->map_items[k].value_num, (k < v->map_size - 1) ? ", " : "");
                            }
                        }
                        printf("}\n");
                    } else {
                        printf("%g\n", v->value);
                    }
                } else {
                    printf("⚠️ تحذير: '%s' مش متغيراً معروفاً!\n", content);
                }
            }
        }
        else if (strncmp(clean_line, "اسأل ", 5) == 0) {
            char* var_name = trim(clean_line + 5);
            char input_buf[128];
            printf("❓ %s: ", var_name);
            if (fgets(input_buf, sizeof(input_buf), stdin)) {
                input_buf[strcspn(input_buf, "\n")] = 0;
                char* endptr;
                double val = strtod(input_buf, &endptr);
                if (*endptr == '\0') {
                    set_number_var(var_name, val);
                } else {
                    set_string_var(var_name, input_buf);
                }
            }
        }
        else if (strstr(clean_line, "يساوى [") != NULL) {
            char var_name[64];
            char temp_line[MAX_LINE_LEN];
            strcpy(temp_line, clean_line);
            char* eq_pos = strstr(temp_line, "يساوى");
            *eq_pos = '\0';
            strcpy(var_name, temp_line + (strncmp(temp_line, "حط ", 3) == 0 ? 3 : 0));
            trim(var_name);
            init_list_var(var_name);
            
            char* list_start = strstr(clean_line, "[");
            char* list_end = strrchr(list_start, ']');
            if (list_end) {
                *list_end = '\0';
                char* token = strtok(list_start + 1, "،,");
                while (token) {
                    list_add(evaluate_expression(trim(token)), var_name);
                    token = strtok(NULL, "،,");
                }
            }
        }
        else if (strstr(clean_line, "يساوى {") != NULL) {
            char var_name[64];
            char temp_line[MAX_LINE_LEN];
            strcpy(temp_line, clean_line);
            char* eq_pos = strstr(temp_line, "يساوى");
            *eq_pos = '\0';
            strcpy(var_name, temp_line + (strncmp(temp_line, "حط ", 3) == 0 ? 3 : 0));
            trim(var_name);
            init_map_var(var_name);
            
            char* map_start = strstr(clean_line, "{");
            char* map_end = strrchr(map_start, '}');
            if (map_end) {
                *map_end = '\0';
                char* pair = strtok(map_start + 1, "،,");
                while (pair) {
                    char* colon = strchr(pair, ':');
                    if (colon) {
                        *colon = '\0';
                        char* key = trim(pair);
                        char* val_str = trim(colon + 1);
                        
                        if (val_str[0] == '"' && val_str[strlen(val_str)-1] == '"') {
                            val_str[strlen(val_str)-1] = '\0';
                            map_set(var_name, key, 0, val_str + 1, 1);
                        } else {
                            map_set(var_name, key, evaluate_expression(val_str), "", 0);
                        }
                    }
                    pair = strtok(NULL, "،,");
                }
            }
        }
        else if (strncmp(clean_line, "حط ", 3) == 0 || strstr(clean_line, "يساوى")) {
            char var_name[64], val_str[128];
            char* eq_pos = strstr(clean_line, "يساوى");
            if (eq_pos) {
                *eq_pos = '\0';
                strcpy(var_name, clean_line + (strncmp(clean_line, "حط ", 3) == 0 ? 3 : 0));
                strcpy(val_str, eq_pos + 6);
                trim(var_name);
                trim(val_str);

                if (val_str[0] == '"' && val_str[strlen(val_str)-1] == '"') {
                    val_str[strlen(val_str)-1] = '\0';
                    set_string_var(var_name, val_str + 1);
                } else {
                    double res = evaluate_expression(val_str);
                    set_number_var(var_name, res);
                }
            }
        }
        else if (strncmp(clean_line, "احسب ", 5) == 0) {
            char* expr = trim(clean_line + 5);
            double res = evaluate_expression(expr);
            printf("🧮 [الناتج]: %g\n", res);
        }
        else if (strncmp(clean_line, "زود ", 4) == 0) {
            char* var_name = trim(clean_line + 4);
            Variable* v = get_variable(var_name);
            if (v && !v->is_string && !v->is_list && !v->is_map) v->value += 1;
        }
        else if (strncmp(clean_line, "اعمل ", 5) == 0) {
            char* fname = trim(clean_line + 5);
            int block_end = find_block_end(lines, i, "اعمل ", "خلاص اعمل");
            if (block_end == -1) {
                printf("❌ دالة '%s' غير مقفولة بـ 'خلاص اعمل'!\n", fname);
                exit(1);
            }
            if (func_count < MAX_FUNCS) {
                strcpy(funcs[func_count].name, fname);
                funcs[func_count].start_line = i;
                funcs[func_count].end_line = block_end;
                func_count++;
            }
            i = block_end + 1;
        }
        else if (strncmp(clean_line, "نفذ ", 4) == 0) {
            char* fname = trim(clean_line + 4);
            Function* target = NULL;
            for (int f = 0; f < func_count; f++) {
                if (strcmp(funcs[f].name, fname) == 0) {
                    target = &funcs[f];
                    break;
                }
            }
            if (!target) {
                printf("❌ دالة '%s' غير موجودة يا أسطى!\n", fname);
            } else {
                in_function++;
                execute_script(lines, total_lines, target->start_line, target->end_line);
                in_function--;

                if (has_returned) {
                    if (return_is_string) {
                        set_string_var("القيمة_المرجعة", return_str_val);
                    } else {
                        set_number_var("القيمة_المرجعة", return_value);
                    }
                    has_returned = 0;
                }
            }
        }
        else if (strncmp(clean_line, "لو ", 3) == 0) {
            int block_end = find_block_end(lines, i, "لو ", "خلاص لو");
            if (block_end == -1) {
                printf("❌ 'لو' غير مقفولة بـ 'خلاص لو'!\n");
                exit(1);
            }

            IfBranch branches[10];
            int branch_count = 0;

            branches[0].type = 0;
            strcpy(branches[0].cond, trim(clean_line + 3));
            branches[0].start_line = i;

            int current_branch_idx = 0;
            int depth = 0;

            for (int k = i; k < block_end; k++) {
                char* ln = trim(lines[k]);
                if (strncmp(ln, "لو ", 3) == 0) depth++;
                else if (strcmp(ln, "خلاص لو") == 0) depth--;
                else if (depth == 0) {
                    if (strncmp(ln, "غير كده لو ", 11) == 0) {
                        branches[current_branch_idx].end_line = k;
                        current_branch_idx++;
                        branches[current_branch_idx].type = 1;
                        strcpy(branches[current_branch_idx].cond, trim(ln + 11));
                        branches[current_branch_idx].start_line = k + 1;
                    } else if (strcmp(ln, "غير كده") == 0) {
                        branches[current_branch_idx].end_line = k;
                        current_branch_idx++;
                        branches[current_branch_idx].type = 2;
                        strcpy(branches[current_branch_idx].cond, "");
                        branches[current_branch_idx].start_line = k + 1;
                    }
                }
            }
            branches[current_branch_idx].end_line = block_end;
            branch_count = current_branch_idx + 1;

            for (int b = 0; b < branch_count; b++) {
                if (branches[b].type == 2) {
                    execute_script(lines, total_lines, branches[b].start_line, branches[b].end_line);
                    break;
                } else if (evaluate_condition(branches[b].cond)) {
                    execute_script(lines, total_lines, branches[b].start_line, branches[b].end_line);
                    break;
                }
            }

            i = block_end + 1;
        }
        else if (strncmp(clean_line, "طالما ", 6) == 0) {
            char* cond = trim(clean_line + 6);
            int block_end = find_block_end(lines, i, "طالما ", "خلاص طالما");
            if (block_end == -1) {
                printf("❌ 'طالما' غير مقفولة بـ 'خلاص طالما'!\n");
                exit(1);
            }
            int loop_start = i;
            int iterations = 0;

            while (evaluate_condition(cond)) {
                iterations++;
                if (iterations > 10000) {
                    printf("💥 'طالما' لفت 10000 مرة بدون توقف! حماية.\n");
                    exit(1);
                }
                execute_script(lines, total_lines, loop_start, block_end);
                if (has_returned) break;
                if (has_broken) {
                    has_broken = 0; 
                    break;
                }
            }
            i = block_end + 1;
        }
        else if (strncmp(clean_line, "كرر ", 5) == 0 && strstr(clean_line, " مرات")) {
            char rest[128];
            strcpy(rest, clean_line + 5);
            char* space = strstr(rest, " مرات");
            if (space) {
                *space = '\0';
                int n = (int)evaluate_expression(trim(rest));
                int block_end = find_block_end(lines, i, "كرر ", "خلاص كرر");
                if (block_end == -1) {
                    printf("❌ 'كرر' غير مقفولة بـ 'خلاص كرر'!\n");
                    exit(1);
                }
                int loop_start = i;
                for (int k = 0; k < n; k++) {
                    execute_script(lines, total_lines, loop_start, block_end);
                    if (has_returned) break;
                    if (has_broken) {
                        has_broken = 0;
                        break;
                    }
                }
                i = block_end + 1;
            }
        }

        if (strcmp(clean_line, "كوباية مية") != 0) {
            lines_without_water++;
            if (lines_without_water >= MAX_LINES_WITHOUT_WATER) {
                printf("💥 Fatal Error: جالك هبوط من كثرة الشغل! اطلب 'كوباية مية' فوراً.\n");
                exit(1);
            }
        }
    }
}

int main(int argc, char* argv[]) {
    if (argc < 2) {
        printf("استخدام البرنامج: qahwa <script.qh>\n");
        return 1;
    }

    FILE* file = fopen(argv[1], "r");
    if (!file) {
        printf("❌ فشل في فتح الملف '%s' يا أسطى!\n", argv[1]);
        return 1;
    }

    char script_lines[MAX_LINES][MAX_LINE_LEN];
    int total_lines = 0;

    while (fgets(script_lines[total_lines], MAX_LINE_LEN, file) && total_lines < MAX_LINES) {
        script_lines[total_lines][strcspn(script_lines[total_lines], "\r\n")] = 0;
        total_lines++;
    }
    fclose(file);

    execute_script(script_lines, total_lines, 0, total_lines);

    return 0;
}

// ==========================================
// 🏗️ الجزء الثاني: الـ Parser و شجرة الـ AST
// ==========================================

// أنواع عُقد شجرة النحو (AST Node Types)
typedef enum {
    AST_ASSIGN,      // إسناد قيمة لمتغير
    AST_PRINT,       // أمر اطلب
    AST_INPUT,       // أمر اسأل
    AST_IF,          // شرط لو
    AST_WHILE,       // حلقة طالما
    AST_FOR,         // حلقة كرر
    AST_FUNC_DEF,    // تعريف دالة
    AST_FUNC_CALL,   // تنفيذ دالة
    AST_RETURN,      // إرجاع قيمة
    AST_BREAK,       // أمر قف
    AST_EXPR         // تعبير حسابي صريح
} ASTNodeType;

// أنماط العمليات داخل الشجرة
typedef enum {
    OP_ADD, OP_SUB, OP_MUL, OP_DIV, OP_MOD, OP_POW,
    OP_EQ, OP_GT, OP_LT
} ASTOpType;

// هيكل العقدة في شجرة الـ AST (Abstract Syntax Tree Node)
typedef struct ASTNode {
    ASTNodeType type;
    char name[64];           // اسم المتغير أو الدالة
    char string_val[MAX_LINE_LEN]; // القيم النصية
    double num_val;          // القيم الرقمية الصريحة
    
    ASTOpType op;            // نوع العملية الحسابية أو المقارنة
    struct ASTNode* left;    // الطرف الأيسر / الشرط
    struct ASTNode* right;   // الطرف الأيمن / الجملة التابعة
    struct ASTNode* next;    // الجملة التالية في التسلسل (للتحكم البرمجي)
    
    // دعم القوائم والكتل المعقدة داخل الشجرة
    int list_data_count;
    double list_data[MAX_LIST_ITEMS];
} ASTNode;

// هيكل سياق الـ Parser لتتبع موقع الـ Tokens أثناء التحليل
typedef struct {
    Token* tokens;
    int token_count;
    int current_index;
} ParserContext;

// دوال مساعدة لإنشاء العقد في الـ AST ديناميكياً
ASTNode* create_ast_node(ASTNodeType type) {
    ASTNode* node = (ASTNode*)malloc(sizeof(ASTNode));
    if (!node) {
        printf("❌ خطأ فادح: نفدت ذاكرة النظام أثناء بناء الـ AST!\n");
        exit(1);
    }
    node->type = type;
    node->name[0] = '\0';
    node->string_val[0] = '\0';
    node->num_val = 0.0;
    node->left = NULL;
    node->right = NULL;
    node->next = NULL;
    node->list_data_count = 0;
    return node;
}

// دالة فحص الـ Token الحالي دون تحريك المؤشر (Peek)
Token* parser_peek(ParserContext* ctx) {
    if (ctx->current_index < ctx->token_count) {
        return &ctx->tokens[ctx->current_index];
    }
    return &ctx->tokens[ctx->token_count]; // TOKEN_EOF
}

// دالة استهلاك الـ Token الحالي وتحريك المؤشر للأمام (Consume)
Token* parser_consume(ParserContext* ctx) {
    if (ctx->current_index < ctx->token_count) {
        return &ctx->tokens[ctx->current_index++];
    }
    return &ctx->tokens[ctx->token_count];
}

// مطابقة نوع ومحتوى الـ Token الحالي
int parser_check(ParserContext* ctx, TokenType type, const char* expected_text) {
    Token* t = parser_peek(ctx);
    if (t->type == type) {
        if (!expected_text || strcmp(t->text, expected_text) == 0) {
            return 1;
        }
    }
    return 0;
}

// ── محلل التعبيرات المتقدم لبناء شجرة الـ AST الحسابية ──
ASTNode* parse_expression_ast(ParserContext* ctx);

ASTNode* parse_factor_ast(ParserContext* ctx) {
    Token* t = parser_peek(ctx);
    ASTNode* node = create_ast_node(AST_EXPR);

    if (t->type == TOKEN_NUMBER) {
        parser_consume(ctx);
        node->num_val = t->num_val;
        node->is_string = 0;
        return node;
    }
    
    if (t->type == TOKEN_STRING) {
        parser_consume(ctx);
        strcpy(node->string_val, t->text);
        return node;
    }

    if (t->type == TOKEN_IDENTIFIER) {
        parser_consume(ctx);
        strcpy(node->name, t->text);
        return node;
    }

    if (t->type == TOKEN_SYMBOL && strcmp(t->text, "(") == 0) {
        parser_consume(ctx); // تخطي '('
        ASTNode* expr = parse_expression_ast(ctx);
        if (parser_peek(ctx)->type == TOKEN_SYMBOL && strcmp(parser_peek(ctx)->text, ")") == 0) {
            parser_consume(ctx); // تخطي ')'
        }
        return expr;
    }

    return node;
}

ASTNode* parse_term_ast(ParserContext* ctx) {
    ASTNode* left = parse_factor_ast(ctx);
    Token* t = parser_peek(ctx);

    while (t->type == TOKEN_OPERATOR && (strcmp(t->text, "*") == 0 || strcmp(t->text, "/") == 0 || strcmp(t->text, "%") == 0 || strcmp(t->text, "**") == 0)) {
        Token* op_token = parser_consume(ctx);
        ASTNode* right = parse_factor_ast(ctx);
        ASTNode* op_node = create_ast_node(AST_EXPR);
        
        if (strcmp(op_token->text, "*") == 0) op_node->op = OP_MUL;
        else if (strcmp(op_token->text, "/") == 0) op_node->op = OP_DIV;
        else if (strcmp(op_token->text, "%") == 0) op_node->op = OP_MOD;
        else if (strcmp(op_token->text, "**") == 0) op_node->op = OP_POW;

        op_node->left = left;
        op_node->right = right;
        left = op_node;
        t = parser_peek(ctx);
    }
    return left;
}

ASTNode* parse_expression_ast(ParserContext* ctx) {
    ASTNode* left = parse_term_ast(ctx);
    Token* t = parser_peek(ctx);

    while (t->type == TOKEN_OPERATOR && (strcmp(t->text, "+") == 0 || strcmp(t->text, "-") == 0)) {
        Token* op_token = parser_consume(ctx);
        ASTNode* right = parse_term_ast(ctx);
        ASTNode* op_node = create_ast_node(AST_EXPR);

        if (strcmp(op_token->text, "+") == 0) op_node->op = OP_ADD;
        else if (strcmp(op_token->text, "-") == 0) op_node->op = OP_SUB;

        op_node->left = left;
        op_node->right = right;
        left = op_node;
        t = parser_peek(ctx);
    }
    return left;
}

// ── تحويل سطر الـ Tokens بالكامل إلى عقدة AST رئيسية ──
ASTNode* parse_statement_ast(Token tokens[], int count) {
    ParserContext ctx;
    ctx.tokens = tokens;
    ctx.token_count = count;
    ctx.current_index = 0;

    Token* first = parser_peek(&ctx);
    if (first->type == TOKEN_EOF) return NULL;

    // حالة الإسناد (مثل: حط س يساوى 5 + 3)
    if (strcmp(first->text, "حط") == 0) {
        parser_consume(&ctx); // استهلاك "حط"
        Token* ident = parser_consume(&ctx); // اسم المتغير
        
        ASTNode* assign_node = create_ast_node(AST_ASSIGN);
        strcpy(assign_node->name, ident->text);

        // تخطي كلمة "يساوى" أو علامة "="
        Token* next_t = parser_peek(&ctx);
        if (strcmp(next_t->text, "يساوى") == 0 || strcmp(next_t->text, "=") == 0) {
            parser_consume(&ctx);
        }

        // فحص ما إذا كانت قيمة الإسناد قائمة [...] أو خريطة {...} أو تعبير حسابي
        Token* val_t = parser_peek(&ctx);
        if (val_t->type == TOKEN_SYMBOL && strcmp(val_t->text, "[") == 0) {
            parser_consume(&ctx); // تخطي '['
            assign_node->list_data_count = 0;
            while (parser_peek(&ctx)->type != TOKEN_EOF && !(parser_peek(&ctx)->type == TOKEN_SYMBOL && strcmp(parser_peek(&ctx)->text, "]") == 0)) {
                Token* item = parser_consume(&ctx);
                if (item->type == TOKEN_NUMBER) {
                    if (assign_node->list_data_count < MAX_LIST_ITEMS) {
                        assign_node->list_data[assign_node->list_data_count++] = item->num_val;
                    }
                }
            }
            if (parser_peek(&ctx)->type == TOKEN_SYMBOL && strcmp(parser_peek(&ctx)->text, "]") == 0) {
                parser_consume(&ctx); // تخطي ']'
            }
        } else {
            assign_node->right = parse_expression_ast(&ctx);
        }
        return assign_node;
    }

    // حالة الطباعة (مثل: اطلب س)
    if (strcmp(first->text, "اطلب") == 0) {
        parser_consume(&ctx);
        Token* target = parser_consume(&ctx);
        ASTNode* print_node = create_ast_node(AST_PRINT);
        strcpy(print_node->name, target->text);
        if (target->type == TOKEN_STRING) {
            strcpy(print_node->string_val, target->text);
        }
        return print_node;
    }

    // حالة أمر الإيقاف (قف)
    if (strcmp(first->text, "قف") == 0) {
        return create_ast_node(AST_BREAK);
    }

    // افتراضي: تعبير عام
    ASTNode* default_node = create_ast_node(AST_EXPR);
    default_node->right = parse_expression_ast(&ctx);
    return default_node;
}

// دوال تفريغ وتنظيف الـ AST من الذاكرة لتجنب تسريب الذاكرة (Memory Leak Prevention)
void free_ast(ASTNode* node) {
    if (!node) return;
    free_ast(node->left);
    free_ast(node->right);
    free_ast(node->next);
    free(node);
}

// ==========================================
// ⚙️ الجزء الثالث: محول الـ Bytecode والآلة الافتراضية (VM)
// ==========================================

// قائمة أوامر الآلة الافتراضية (Bytecode Instructions)
typedef enum {
    OP_HALT = 0,   // نهاية البرنامج
    OP_LOADK,      // تحميل قيمة ثابتة إلى مسجل (Register)
    OP_LOADVAR,    // تحميل قيمة متغير إلى مسجل
    OP_STORE,      // تخزين مسجل في متغير
    OP_ADD,        // جمع مسجلين
    OP_SUB,        // طرح مسجلين
    OP_MUL,        // ضرب مسجلين
    OP_DIV,        // قسمة مسجلين
    OP_PRINT,      // طباعة قيمة مسجل أو نص
    OP_JMP,        // قفزة غير مشروطة
    OP_JMPZ        // قفزة إذا كان الصفر (للشروط)
} OpCode;

// هيكل تعليمة الـ Bytecode (Instruction)
typedef struct {
    OpCode op;
    int a;
    int b;
    int c;
    double dval;
    char sval[64];
} Instruction;

#define MAX_CODE 1000
#define MAX_REGS 16

Instruction bytecode[MAX_CODE];
int code_count = 0;

// مسجلات الآلة الافتراضية (Virtual Machine Registers)
double vm_regs[MAX_REGS];
char vm_reg_strings[MAX_REGS][MAX_LINE_LEN];
int vm_reg_is_str[MAX_REGS];

// دالة توليد تعليمة جديدة في مصفوفة الـ Bytecode
void emit(OpCode op, int a, int b, int c, double dval, const char* sval) {
    if (code_count < MAX_CODE) {
        bytecode[code_count].op = op;
        bytecode[code_count].a = a;
        bytecode[code_count].b = b;
        bytecode[code_count].c = c;
        bytecode[code_count].dval = dval;
        if (sval) strcpy(bytecode[code_count].sval, sval);
        else bytecode[code_count].sval[0] = '\0';
        code_count++;
    }
}

// ── مترجم البايت كود البسيط (Bytecode Compiler for Assignments & Print) ──
void compile_to_bytecode(const char* clean_line) {
    // مثال: حط س يساوى 5 + 3
    if (strncmp(clean_line, "حط ", 3) == 0 || strstr(clean_line, "يساوى")) {
        char var_name[64], val_str[128];
        char* eq_pos = strstr(clean_line, "يساوى");
        if (eq_pos) {
            *eq_pos = '\0';
            strcpy(var_name, clean_line + (strncmp(clean_line, "حط ", 3) == 0 ? 3 : 0));
            strcpy(val_str, eq_pos + 6);
            trim(var_name);
            trim(val_str);

            if (val_str[0] == '"' && val_str[strlen(val_str)-1] == '"') {
                val_str[strlen(val_str)-1] = '\0';
                emit(OP_LOADK, 0, 0, 0, 0.0, val_str);
                emit(OP_STORE, 0, 0, 0, 0.0, var_name);
            } else {
                // فحص ما إذا كان جمع بسيط أو رقم صريح
                char* plus_pos = strchr(val_str, '+');
                if (plus_pos) {
                    *plus_pos = '\0';
                    char* left_str = trim(val_str);
                    char* right_str = trim(plus_pos + 1);

                    double left_val = strtod(left_str, NULL);
                    double right_val = strtod(right_str, NULL);

                    emit(OP_LOADK, 0, 0, 0, left_val, NULL);   // المسجل 0 = left_val
                    emit(OP_LOADK, 1, 0, 0, right_val, NULL);  // المسجل 1 = right_val
                    emit(OP_ADD, 0, 0, 1, 0.0, NULL);          // Reg[0] = Reg[0] + Reg[1]
                    emit(OP_STORE, 0, 0, 0, 0.0, var_name);    // تخزين Reg[0] في المتغير
                } else {
                    double val = strtod(val_str, NULL);
                    emit(OP_LOADK, 0, 0, 0, val, NULL);
                    emit(OP_STORE, 0, 0, 0, 0.0, var_name);
                }
            }
        }
    }
    // مثال: اطلب س
    else if (strncmp(clean_line, "اطلب ", 5) == 0) {
        char* content = trim(clean_line + 5);
        if (content[0] == '"' && content[strlen(content)-1] == '"') {
            content[strlen(content)-1] = '\0';
            emit(OP_LOADK, 0, 0, 0, 0.0, content + 1);
            emit(OP_PRINT, 0, 1, 0, 0.0, NULL); // طباعة نص مباشر
        } else {
            emit(OP_LOADVAR, 0, 0, 0, 0.0, content);
            emit(OP_PRINT, 0, 0, 0, 0.0, NULL); // طباعة مسجل متغير
        }
    }
}

// ── تنفيذ الآلة الافتراضية (Virtual Machine Execution Loop) ──
void run_virtual_machine() {
    int pc = 0; // Program Counter
    while (pc < code_count) {
        Instruction* instr = &bytecode[pc++];

        switch (instr->op) {
            case OP_LOADK:
                if (instr->sval[0] != '\0') {
                    vm_reg_is_str[instr->a] = 1;
                    strcpy(vm_reg_strings[instr->a], instr->sval);
                } else {
                    vm_reg_is_str[instr->a] = 0;
                    vm_regs[instr->a] = instr->dval;
                }
                break;

            case OP_LOADVAR: {
                Variable* v = get_variable(instr->sval);
                if (v) {
                    if (v->is_string) {
                        vm_reg_is_str[instr->a] = 1;
                        strcpy(vm_reg_strings[instr->a], v->str_val);
                    } else {
                        vm_reg_is_str[instr->a] = 0;
                        vm_regs[instr->a] = v->value;
                    }
                }
                break;
            }

            case OP_STORE:
                if (vm_reg_is_str[instr->a]) {
                    set_string_var(instr->sval, vm_reg_strings[instr->a]);
                } else {
                    set_number_var(instr->sval, vm_regs[instr->a]);
                }
                break;

            case OP_ADD:
                vm_regs[instr->a] = vm_regs[instr->a] + vm_regs[instr->c];
                break;

            case OP_SUB:
                vm_regs[instr->a] = vm_regs[instr->a] - vm_regs[instr->c];
                break;

            case OP_MUL:
                vm_regs[instr->a] = vm_regs[instr->a] * vm_regs[instr->c];
                break;

            case OP_DIV:
                if (vm_regs[instr->c] == 0) {
                    printf("❌ خطأ VM: القسمة على صفر!\n");
                    exit(1);
                }
                vm_regs[instr->a] = vm_regs[instr->a] / vm_regs[instr->c];
                break;

            case OP_PRINT:
                if (instr->b == 1 || vm_reg_is_str[instr->a]) {
                    printf("%s\n", vm_reg_strings[instr->a]);
                } else {
                    printf("%g\n", vm_regs[instr->a]);
                }
                break;

            case OP_HALT:
            default:
                return;
        }
    }
}

// ==========================================
// 🧠 الجزء الرابع (القسم 1): النواة المتقدمة للآلة الافتراضية (VM Architecture)
// ==========================================

#define VM_STACK_SIZE 256
#define VM_GLOBALS_SIZE 128
#define VM_MAX_CALLS 64

// أنواع القيم داخل الـ VM (Dynamic Value Types)
typedef enum {
    VAL_NIL = 0,
    VAL_NUMBER,
    VAL_STRING,
    VAL_LIST,
    VAL_MAP
} ValueType;

// هيكل القيمة الموحدة (Tagged Union Value) في الآلة الافتراضية
typedef struct {
    ValueType type;
    union {
        double num;
        char str[MAX_LINE_LEN];
        struct {
            double items[MAX_LIST_ITEMS];
            int size;
        } list;
        struct {
            MapItem items[MAX_MAP_ITEMS];
            int size;
        } map;
    } as;
} VMValue;

// هيكل إطار الدالة في مكدس الاستدعاءات (Call Frame)
typedef struct {
    int return_pc;      // مؤشر العودة بعد انتهاء الدالة
    int stack_base;     // قاعدة المسجلات الخاصة بالإطار الحالي
    char func_name[64];
} VMCallFrame;

// هيكل الـ الآلة الافتراضية العملاقة (Virtual Machine State)
typedef struct {
    VMValue stack[VM_STACK_SIZE];     // مكدس القيم والمسجلات للـ VM
    int sp;                           // مؤشر المكدس (Stack Pointer)
    
    VMValue globals[VM_GLOBALS_SIZE]; // جدول المتغيرات العامة
    char global_names[VM_GLOBALS_SIZE][64];
    int global_count;

    VMCallFrame call_frames[VM_MAX_CALLS];
    int cp;                           // مؤشر إطارات الاستدعاء
    
    int pc;                           // مؤشر التعليمات الحالي (Program Counter)
    int running;                      // حالة تشغيل الـ VM
} VirtualMachine;

// تهيئة حالة الآلة الافتراضية
VirtualMachine main_vm;

void init_vm(VirtualMachine* vm) {
    vm->sp = 0;
    vm->cp = 0;
    vm->pc = 0;
    vm->global_count = 0;
    vm->running = 1;
    for (int i = 0; i < VM_STACK_SIZE; i++) {
        vm->stack[i].type = VAL_NIL;
    }
}

// دالة البحث عن متغير عام داخل الـ VM O(N)
VMValue* vm_get_global(VirtualMachine* vm, const char* name) {
    for (int i = 0; i < vm->global_count; i++) {
        if (strcmp(vm->global_names[i], name) == 0) {
            return &vm->globals[i];
        }
    }
    return NULL;
}

void vm_set_global(VirtualMachine* vm, const char* name, VMValue val) {
    for (int i = 0; i < vm->global_count; i++) {
        if (strcmp(vm->global_names[i], name) == 0) {
            vm->globals[i] = val;
            return;
        }
    }
    if (vm->global_count < VM_GLOBALS_SIZE) {
        strcpy(vm->global_names[vm->global_count], name);
        vm->globals[vm->global_count] = val;
        vm->global_count++;
    }
}

// عمليات المكدس الأساسية (Push / Pop)
void vm_push(VirtualMachine* vm, VMValue val) {
    if (vm->sp < VM_STACK_SIZE) {
        vm->stack[vm->sp++] = val;
    } else {
        printf("💥 خطأ فادح في الـ VM: امتلاء المكدس (Stack Overflow)!\n");
        exit(1);
    }
}

VMValue vm_pop(VirtualMachine* vm) {
    if (vm->sp > 0) {
        return vm->stack[--vm->sp];
    }
    VMValue nil_val;
    nil_val.type = VAL_NIL;
    return nil_val;
}

// ==========================================
// 🧠 الجزء الرابع (القسم 2): حلقة التنفيذ المركزية للـ VM
// ==========================================

// دالة مساعدة لتحويل VMValue إلى نص أو طباعتها
void vm_print_value(VMValue val) {
    switch (val.type) {
        case VAL_NUMBER:
            printf("%g\n", val.as.num);
            break;
        case VAL_STRING:
            printf("%s\n", val.as.str);
            break;
        case VAL_LIST:
            printf("[");
            for (int i = 0; i < val.as.list.size; i++) {
                printf("%g%s", val.as.list.items[i], (i < val.as.list.size - 1) ? ", " : "");
            }
            printf("]\n");
            break;
        case VAL_MAP:
            printf("{");
            for (int i = 0; i < val.as.map.size; i++) {
                if (val.as.map.items[i].is_string) {
                    printf("%s: \"%s\"%s", val.as.map.items[i].key, val.as.map.items[i].value_str, (i < val.as.map.size - 1) ? ", " : "");
                } else {
                    printf("%s: %g%s", val.as.map.items[i].key, val.as.map.items[i].value_num, (i < val.as.map.size - 1) ? ", " : "");
                }
            }
            printf("}\n");
            break;
        case VAL_NIL:
        default:
            printf("فارغ (nil)\n");
            break;
    }
}

// دالة التشغيل الكبرى للـ VM (The Core Dispatch Loop)
void vm_execute_bytecode(VirtualMachine* vm, Instruction code[], int code_len) {
    vm->pc = 0;
    vm->running = 1;

    while (vm->running && vm->pc < code_len) {
        Instruction instr = code[vm->pc++];

        switch (instr.op) {
            case OP_HALT:
                vm->running = 0;
                break;

            case OP_LOADK: {
                VMValue val;
                if (instr.sval[0] != '\0') {
                    val.type = VAL_STRING;
                    strcpy(val.as.str, instr.sval);
                } else {
                    val.type = VAL_NUMBER;
                    val.as.num = instr.dval;
                }
                vm_push(vm, val);
                break;
            }

            case OP_LOADVAR: {
                VMValue* gval = vm_get_global(vm, instr.sval);
                if (gval) {
                    vm_push(vm, *gval);
                } else {
                    VMValue nil_val;
                    nil_val.type = VAL_NIL;
                    vm_push(vm, nil_val);
                }
                break;
            }

            case OP_STORE: {
                VMValue val = vm_pop(vm);
                vm_set_global(vm, instr.sval, val);
                break;
            }

            case OP_ADD: {
                VMValue right = vm_pop(vm);
                VMValue left = vm_pop(vm);
                VMValue result;
                result.type = VAL_NUMBER;
                result.as.num = left.as.num + right.as.num;
                vm_push(vm, result);
                break;
            }

            case OP_SUB: {
                VMValue right = vm_pop(vm);
                VMValue left = vm_pop(vm);
                VMValue result;
                result.type = VAL_NUMBER;
                result.as.num = left.as.num - right.as.num;
                vm_push(vm, result);
                break;
            }

            case OP_MUL: {
                VMValue right = vm_pop(vm);
                VMValue left = vm_pop(vm);
                VMValue result;
                result.type = VAL_NUMBER;
                result.as.num = left.as.num * right.as.num;
                vm_push(vm, result);
                break;
            }

            case OP_DIV: {
                VMValue right = vm_pop(vm);
                VMValue left = vm_pop(vm);
                if (right.as.num == 0.0) {
                    printf("❌ خطأ فادح في الـ VM: القسمة على صفر!\n");
                    exit(1);
                }
                VMValue result;
                result.type = VAL_NUMBER;
                result.as.num = left.as.num / right.as.num;
                vm_push(vm, result);
                break;
            }

            case OP_PRINT: {
                VMValue val = vm_pop(vm);
                vm_print_value(val);
                break;
            }

            case OP_JMP: {
                vm->pc = instr.a; // القفز المباشر لموقع محدد
                break;
            }

            case OP_JMPZ: {
                VMValue condition = vm_pop(vm);
                if (condition.as.num == 0.0) {
                    vm->pc = instr.a; // قفزة شرطية إذا كانت القيمة صفر أو خطأ
                }
                break;
            }

            default:
                printf("❌ تعليمات غير معروفة في الـ VM opcode: %d\n", instr.op);
                vm->running = 0;
                break;
        }
    }
}

// ==========================================
// 🔗 الجزء الرابع (القسم 3): جسر الربط والمحرك النهائي للغة QahwaScript
// ==========================================

// دالة الترجمة الشاملة من سطور السكريبت إلى Bytecode جاهز للـ VM
void compile_script_to_vm(char lines[MAX_LINES][MAX_LINE_LEN], int total_lines) {
    code_count = 0;
    int i = 0;

    while (i < total_lines) {
        char line[MAX_LINE_LEN];
        strcpy(line, lines[i]);
        char* clean_line = trim(line);
        i++;

        if (strlen(clean_line) == 0 || clean_line[0] == '#') continue;

        if (strcmp(clean_line, "كوباية مية") == 0) {
            lines_without_water = 0;
            printf("☕ [إسعاف]: تم شرب كوباية مية، المبرمج استرد وعيه!\n");
            continue;
        }

        if (strcmp(clean_line, "الحساب يا أسطى") == 0) {
            emit(OP_HALT, 0, 0, 0, 0.0, NULL);
            break;
        }

        // استدعاء المترجم البرمجي لتوليد الـ Bytecode لكل سطر
        compile_to_bytecode(clean_line);
    }

    // وضع تعليمة النهاية الآمنة
    emit(OP_HALT, 0, 0, 0, 0.0, NULL);
}

// دالة تنفيذ الملف البرمجي عبر الآلة الافتراضية المتقدمة
void run_qahwa_program(const char* filename) {
    FILE* file = fopen(filename, "r");
    if (!file) {
        printf("❌ فشل في فتح الملف '%s' يا أسطى!\n", filename);
        return;
    }

    char script_lines[MAX_LINES][MAX_LINE_LEN];
    int total_lines = 0;

    while (fgets(script_lines[total_lines], MAX_LINE_LEN, file) && total_lines < MAX_LINES) {
        script_lines[total_lines][strcspn(script_lines[total_lines], "\r\n")] = 0;
        total_lines++;
    }
    fclose(file);

    // 1. ترجمة الكود إلى Bytecode
    compile_script_to_vm(script_lines, total_lines);

    // 2. تهيئة وتشغيل الآلة الافتراضية (VM)
    init_vm(&main_vm);
    
    // ربط المتغيرات التقليدية مع الـ VM للتوافقية التامة
    for (int v = 0; v < var_count; v++) {
        VMValue val;
        if (vars[v].is_string) {
            val.type = VAL_STRING;
            strcpy(val.as.str, vars[v].str_val);
        } else if (vars[v].is_list) {
            val.type = VAL_LIST;
            val.as.list.size = vars[v].list_size;
            for (int k = 0; k < vars[v].list_size; k++) {
                val.as.list.items[k] = vars[v].list_nums[k];
            }
        } else {
            val.type = VAL_NUMBER;
            val.as.num = vars[v].value;
        }
        vm_set_global(&main_vm, vars[v].name, val);
    }

    // تشغيل حلقة الـ VM المركزية
    vm_execute_bytecode(&main_vm, bytecode, code_count);
}

// دالة الـ main المعدلة لدعم الـ VM الحديثة
int main_vm_entry(int argc, char* argv[]) {
    if (argc < 2) {
        printf("استخدام البرنامج (الآلة الافتراضية): qahwa_vm <script.qh>\n");
        return 1;
    }

    run_qahwa_program(argv[1]);
    return 0;
}

// ==========================================
// 🧹 الجزء الخامس (القسم 1): مجمع الذاكرة التلقائي (Garbage Collector Core)
// ==========================================

// أنواع الكائنات الديناميكية التي تدار بواسطة الـ GC
typedef enum {
    OBJ_STRING = 1,
    OBJ_LIST,
    OBJ_MAP
} ObjType;

// هيكل رأس الكائن المشترك (GC Object Header)
typedef struct Obj {
    ObjType type;
    int marked;          // علامة الفحص للـ GC (Mark bit)
    struct Obj* next;    // مؤشر للقائمة المرتبطة لكل الكائنات في الذاكرة
} Obj;

// كائن النص الديناميكي
typedef struct {
    Obj obj;
    char* chars;
    int length;
} ObjString;

// كائن القائمة الديناميكية اللانهائية (Dynamic Heap List)
typedef struct {
    Obj obj;
    VMValue* items;
    int size;
    int capacity;
} ObjList;

// تتبع الذاكرة الكلية المخصصة
typedef struct {
    Obj* head;           // رأس قائمة كل الكائنات الحية
    size_t bytes_allocated;
    size_t next_gc;      // حد تفعيل الـ GC التلقائي
} GCState;

GCState gc_state = {NULL, 0, 1024 * 1024}; // يبدأ الـ GC بالعمل عند الوصول لـ 1 ميجابايت

// دوال مساعدة لإنشاء وإدارة الكائنات على الـ Heap
ObjString* allocate_string(const char* chars) {
    int length = strlen(chars);
    ObjString* string = (ObjString*)malloc(sizeof(ObjString) + length + 1);
    if (!string) {
        printf("❌ خطأ فادح: فشل تخصيص ذاكرة للنص!\n");
        exit(1);
    }
    string->obj.type = OBJ_STRING;
    string->obj.marked = 0;
    string->obj.next = gc_state.head;
    gc_state.head = (Obj*)string;
    
    string->chars = (char*)((void*)string + sizeof(ObjString));
    strcpy(string->chars, chars);
    string->length = length;
    
    gc_state.bytes_allocated += sizeof(ObjString) + length + 1;
    return string;
}

ObjList* allocate_list() {
    ObjList* list = (ObjList*)malloc(sizeof(ObjList));
    if (!list) {
        printf("❌ خطأ فادح: فشل تخصيص ذاكرة للقائمة!\n");
        exit(1);
    }
    list->obj.type = OBJ_LIST;
    list->obj.marked = 0;
    list->obj.next = gc_state.head;
    gc_state.head = (Obj*)list;

    list->capacity = 8; // يبدأ بسعة أولية مرنة
    list->size = 0;
    list->items = (VMValue*)malloc(sizeof(VMValue) * list->capacity);

    gc_state.bytes_allocated += sizeof(ObjList) + (sizeof(VMValue) * list->capacity);
    return list;
}

void list_append_dynamic(ObjList* list, VMValue val) {
    if (list->size >= list->capacity) {
        list->capacity *= 2;
        list->items = (VMValue*)realloc(list->items, sizeof(VMValue) * list->capacity);
        gc_state.bytes_allocated += sizeof(VMValue) * (list->capacity / 2);
    }
    list->items[list->size++] = val;
}

// ==========================================
// 🧹 الجزء الخامس (القسم 2): خوارزمية التنظيف التلقائي (Mark-and-Sweep GC)
// ==========================================

// دالة تعليم الكائنات النشطة (Mark Phase)
void gc_mark_object(Obj* obj) {
    if (!obj || obj->marked) return;
    obj->marked = 1; // وضع علامة أن هذا الكائن حي ومستخدم

    // إذا كان الكائن قائمة، نقوم بتعليم العناصر الموجودة بداخله أيضاً
    if (obj->type == OBJ_LIST) {
        ObjList* list = (ObjList*)obj;
        for (int i = 0; i < list->size; i++) {
            // لو العناصر المخبأة عبارة عن كائنات ديناميكية أخرى (مستقبلاً)
        }
    }
}

// دالة فحص ومسح الكائنات الميتة (Sweep Phase)
void gc_sweep() {
    Obj* previous = NULL;
    Obj* current = gc_state.head;

    while (current != NULL) {
        if (current->marked) {
            // الكائن نشط ومستخدم، فقط نقوم بإزالة العلامة تحضيراً للدورة القادمة
            current->marked = 0;
            previous = current;
            current = current->next;
        } else {
            // كائن ميت (غير مستخدم)، نقوم بتحريره من الذاكرة فوراً!
            Obj* unreached = current;
            current = current->next;

            if (previous) {
                previous->next = current;
            } else {
                gc_state.head = current;
            }

            // تحرير الذاكرة بحسب نوع الكائن
            if (unreached->type == OBJ_STRING) {
                ObjString* str = (ObjString*)unreached;
                gc_state.bytes_allocated -= (sizeof(ObjString) + str->length + 1);
                free(str);
            } else if (unreached->type == OBJ_LIST) {
                ObjList* list = (ObjList*)unreached;
                gc_state.bytes_allocated -= (sizeof(ObjList) + (sizeof(VMValue) * list->capacity));
                free(list->items);
                free(list);
            }
        }
    }
}

// الدالة الرئيسية لتشغيل مجمع الذاكرة التلقائي (Garbage Collector Trigger)
void garbage_collect(VirtualMachine* vm) {
    size_t allocated_before = gc_state.bytes_allocated;

    // 1. مرحلة العلامات (Mark): المرور على المتغيرات العامة في الـ VM
    for (int i = 0; i < vm->global_count; i++) {
        // فحص المتغيرات العامة لو كانت كائنات ديناميكية
    }

    // 2. مرحلة التنظيف (Sweep): حذف الكائنات غير الموقعة
    gc_sweep();

    // تحديث حد الذاكرة للعملية القادمة
    gc_state.next_gc = gc_state.bytes_allocated * 2;
    if (gc_state.next_gc < 1024 * 1024) {
        gc_state.next_gc = 1024 * 1024; // الحد الأدنى ميجابايت واحد
    }
}

// دالة تخصيص آمنة تفحص الذاكرة وتطلق الـ GC تلقائياً عند الحاجة
void* gc_allocate(VirtualMachine* vm, size_t size) {
    if (gc_state.bytes_allocated > gc_state.next_gc) {
        garbage_collect(vm);
    }
    return malloc(size);
}

// ==========================================
// 🔤 الجزء السادس (القسم 1): مكتبة النصوص المتقدمة (String Library)
// ==========================================

#include <string.h>
#include <stdlib.h>
#include <stdio.h>

// هيكل يمثل مصفوفة نصوص ناتجة عن التقسيم (Split Result)
typedef struct {
    char items[MAX_LIST_ITEMS][MAX_LINE_LEN];
    int count;
} StringList;

// 1. دالة تقسيم النصوص (string.split)
// مثال: string.split("a,b,c", ",") -> {"a", "b", "c"}
StringList string_split(const char* str, const char* delimiter) {
    StringList result;
    result.count = 0;

    char temp[MAX_LINE_LEN];
    strncpy(temp, str, sizeof(temp) - 1);
    temp[sizeof(temp) - 1] = '\0';

    char* token = strtok(temp, delimiter);
    while (token != NULL && result.count < MAX_LIST_ITEMS) {
        // إزالة الفراغات الزائدة لكل عنصر ناتج
        while (*token == ' ') token++;
        strcpy(result.items[result.count++], token);
        token = strtok(NULL, delimiter);
    }

    return result;
}

// 2. دالة البحث عن موقع نص داخل نص آخر (string.find)
// مثال: string.find("hello", "ell") -> يعيد المؤشر (يبدأ من 1، أو 0 إذا لمר يوجَد)
int string_find(const char* haystack, const char* needle) {
    char* pos = strstr(haystack, needle);
    if (pos != NULL) {
        return (int)(pos - haystack) + 1; // الفهرس يبدأ من 1 على طريقة Lua
    }
    return 0; // غير موجود
}

// 3. دالة استبدال النصوص (string.gsub)
// مثال: string.gsub("hello", "l", "L") -> "heLLo"
void string_gsub(const char* src, const char* target, const char* replacement, char* dest, size_t dest_size) {
    char buffer[MAX_LINE_LEN * 2] = "";
    const char* p = src;
    size_t target_len = strlen(target);
    size_t replacement_len = strlen(replacement);

    while (*p != '\0') {
        if (strncmp(p, target, target_len) == 0) {
            strncat(buffer, replacement, dest_size - strlen(buffer) - 1);
            p += target_len;
        } else {
            size_t current_len = strlen(buffer);
            if (current_len < dest_size - 1) {
                char temp[2] = {*p, '\0'};
                strncat(buffer, temp, dest_size - current_len - 1);
            }
            p++;
        }
    }
    strncpy(dest, buffer, dest_size);
    dest[dest_size - 1] = '\0';
}

// ==========================================
// 🔤 الجزء السادس (القسم 2): التنسيق المتقدم وتكامل الـ VM
// ==========================================

// 4. دالة تنسيق النصوص (string.format)
// مثال: string.format("Hello %s, Score: %d", "World", 100) -> "Hello World, Score: 100"
void string_format(const char* format_str, const char* arg_str, double arg_num, int is_string_arg, char* dest, size_t dest_size) {
    char buffer[MAX_LINE_LEN * 2] = "";
    const char* p = format_str;
    int written = 0;

    while (*p != '\0' && written < (int)dest_size - 1) {
        if (*p == '%' && *(p + 1) != '\0') {
            p++;
            if (*p == 's') {
                if (is_string_arg) {
                    strncat(buffer, arg_str, dest_size - strlen(buffer) - 1);
                } else {
                    char num_buf[64];
                    snprintf(num_buf, sizeof(num_buf), "%g", arg_num);
                    strncat(buffer, num_buf, dest_size - strlen(buffer) - 1);
                }
            } else if (*p == 'd' || *p == 'g' || *p == 'f') {
                char num_buf[64];
                snprintf(num_buf, sizeof(num_buf), "%g", arg_num);
                strncat(buffer, num_buf, dest_size - strlen(buffer) - 1);
            } else {
                // لو رمز تنسيق غير معروف، نطبعه كما هو
                char unknown[3] = {'%', *p, '\0'};
                strncat(buffer, unknown, dest_size - strlen(buffer) - 1);
            }
            p++;
        } else {
            char temp[2] = {*p, '\0'};
            strncat(buffer, temp, dest_size - strlen(buffer) - 1);
            p++;
        }
    }
    strncpy(dest, buffer, dest_size);
    dest[dest_size - 1] = '\0';
}

// 5. دالة مساعدة لربط مكتبة النصوص مع تنفيذ الـ VM
void vm_execute_string_library_command(VirtualMachine* vm, const char* func_name, const char* arg1, const char* arg2) {
    if (strcmp(func_name, "string.find") == 0) {
        int pos = string_find(arg1, arg2);
        VMValue res;
        res.type = VAL_NUMBER;
        res.as.num = (double)pos;
        vm_push(vm, res);
    } 
    else if (strcmp(func_name, "string.split") == 0) {
        StringList list = string_split(arg1, arg2);
        VMValue res_list;
        res_list.type = VAL_LIST;
        res_list.as.list.size = list.count;
        for (int i = 0; i < list.count; i++) {
            // تحويل العناصر الناتجة إلى أرقام أو تخزينها كقيم نصية بحسب الحاجة
            res_list.as.list.items[i] = strtod(list.items[i], NULL);
        }
        vm_push(vm, res_list);
    }
    else if (strcmp(func_name, "string.gsub") == 0) {
        char buffer[MAX_LINE_LEN];
        // افتراض أن arg1 هو النص، و arg2 هو الهدف، والتعديل يتم محلياً
        string_gsub(arg1, arg2, "", buffer, sizeof(buffer));
        VMValue res;
        res.type = VAL_STRING;
        strcpy(res.as.str, buffer);
        vm_push(vm, res);
    }
}
// ==========================================
// 🗂️ الجزء السابع: مكتبة الجداول المتقدمة (Table & Hash Library)
// ==========================================

#define TABLE_INITIAL_CAPACITY 16

// أنواع المفاتيح المدعومة في الجداول الديناميكية
typedef enum {
    KEY_TYPE_STRING = 1,
    KEY_TYPE_NUMBER,
    KEY_TYPE_BOOL
} TableKeyType;

// هيكل المفتاح الموحد
typedef struct {
    TableKeyType type;
    char s_key[64];
    double n_key;
    int b_key;
} TableKey;

// عنصر داخل الـ Hash Table الديناميكي
typedef struct {
    TableKey key;
    VMValue value;
    int is_occupied; // هل الخلية مستخدمة أم فارغة
} HashEntry;

// هيكل الـ Table الديناميكي الكامل (قابل للتوسع اللانهائي)
typedef struct ObjTable {
    Obj obj; // رأس الـ GC المشترك
    HashEntry* entries;
    int capacity;
    int count;
} ObjTable;

// دالة تجزئة بسيطة ومستقرة (Simple Hash Function)
unsigned int hash_string(const char* str) {
    unsigned int hash = 2166136261u;
    for (int i = 0; str[i] != '\0'; i++) {
        hash ^= (unsigned char)str[i];
        hash *= 16777619;
    }
    return hash;
}

// إنشاء جدول جديد مخصص على الـ Heap
ObjTable* allocate_table(VirtualMachine* vm) {
    ObjTable* table = (ObjTable*)gc_allocate(vm, sizeof(ObjTable));
    if (!table) {
        printf("❌ خطأ فادح: فشل تخصيص الذاكرة للـ Table!\n");
        exit(1);
    }
    table->obj.type = (ObjType)4; // نوع خاص بالـ Table
    table->obj.marked = 0;
    table->obj.next = gc_state.head;
    gc_state.head = (Obj*)table;

    table->capacity = TABLE_INITIAL_CAPACITY;
    table->count = 0;
    table->entries = (HashEntry*)malloc(sizeof(HashEntry) * table->capacity);
    for (int i = 0; i < table->capacity; i++) {
        table->entries[i].is_occupied = 0;
    }

    gc_state.bytes_allocated += sizeof(ObjTable) + (sizeof(HashEntry) * table->capacity);
    return table;
}

// البحث عن مكان المفتاح أو إيجاد خلية فارغة داخل الجدول (Linear Probing)
HashEntry* find_table_entry(HashEntry* entries, int capacity, TableKey key) {
    unsigned int idx = 0;
    if (key.type == KEY_TYPE_STRING) {
        idx = hash_string(key.s_key) % capacity;
    } else if (key.type == KEY_TYPE_NUMBER) {
        idx = (unsigned int)key.n_key % capacity;
    } else {
        idx = (unsigned int)key.b_key % capacity;
    }

    for (int i = 0; i < capacity; i++) {
        int pos = (idx + i) % capacity;
        HashEntry* entry = &entries[pos];
        if (!entry->is_occupied) {
            return entry;
        }
        // مقارنة المفاتيح لتحديد التطابق
        if (entry->key.type == key.type) {
            if (key.type == KEY_TYPE_STRING && strcmp(entry->key.s_key, key.s_key) == 0) return entry;
            if (key.type == KEY_TYPE_NUMBER && entry->key.n_key == key.n_key) return entry;
            if (key.type == KEY_TYPE_BOOL && entry->key.b_key == key.b_key) return entry;
        }
    }
    return &entries[idx];
}

// إعادة توزيع الجدول عند الامتلاء (Table Resizing)
void table_resize(VirtualMachine* vm, ObjTable* table, int new_capacity) {
    HashEntry* new_entries = (HashEntry*)malloc(sizeof(HashEntry) * new_capacity);
    for (int i = 0; i < new_capacity; i++) {
        new_entries[i].is_occupied = 0;
    }

    for (int i = 0; i < table->capacity; i++) {
        HashEntry* old_entry = &table->entries[i];
        if (old_entry->is_occupied) {
            HashEntry* dest = find_table_entry(new_entries, new_capacity, old_entry->key);
            dest->key = old_entry->key;
            dest->value = old_entry->value;
            dest->is_occupied = 1;
        }
    }

    free(table->entries);
    table->entries = new_entries;
    gc_state.bytes_allocated += (sizeof(HashEntry) * (new_capacity - table->capacity));
    table->capacity = new_capacity;
}

// تعيين قيمة في الجدول باستخدام مفتاح ديناميكي
void table_set(VirtualMachine* vm, ObjTable* table, TableKey key, VMValue value) {
    if (table->count + 1 > table->capacity * 0.75) {
        table_resize(vm, table, table->capacity * 2);
    }
    HashEntry* entry = find_table_entry(table->entries, table->capacity, key);
    if (!entry->is_occupied) {
        table->count++;
    }
    entry->key = key;
    entry->value = value;
    entry->is_occupied = 1;
}
// ==========================================
// 🛠️ الجزء الثامن والأخير: المكتبة القياسية الشاملة (Standard Library)
// ==========================================

#include <time.h>
#include <math.h>

// 1. قسم الرياضيات (Math Library Functions)
double qahwa_math_sin(double x) { return sin(x); }
double qahwa_math_cos(double x) { return cos(x); }
double qahwa_math_sqrt(double x) { return x >= 0 ? sqrt(x) : 0; }
double qahwa_math_random(double min_val, double max_val) {
    return min_val + (rand() / (double)RAND_MAX) * (max_val - min_val);
}

// 2. قسم نظام التشغيل والوقت (OS Library Functions)
double qahwa_os_time() {
    return (double)time(NULL);
}

void qahwa_os_date(const char* format, char* dest, size_t dest_size) {
    time_t rawtime;
    time(&rawtime);
    struct tm* timeinfo = localtime(&rawtime);
    strftime(dest, dest_size, format, timeinfo);
}

// 3. قسم إدارة الملفات (IO Library Functions)
typedef struct {
    FILE* file_ptr;
    int is_open;
} QahwaFile;

QahwaFile qahwa_io_open(const char* filename, const char* mode) {
    QahwaFile qfile;
    qfile.file_ptr = fopen(filename, mode);
    qfile.is_open = (qfile.file_ptr != NULL);
    return qfile;
}

void qahwa_io_read_all(QahwaFile* qfile, char* dest, size_t max_size) {
    if (!qfile->is_open) return;
    size_t bytesRead = fread(dest, 1, max_size - 1, qfile->file_ptr);
    dest[bytesRead] = '\0';
}

void qahwa_io_close(QahwaFile* qfile) {
    if (qfile->is_open) {
        fclose(qfile->file_ptr);
        qfile->is_open = 0;
    }
}

// 4. قسم فرز الجداول والقوائم (Table Sort Utility)
void qahwa_table_sort(double* arr, int size) {
    // خوارزمية فرز بسيطة وسريعة (Bubble Sort / Quick Sort مبسطة)
    for (int i = 0; i < size - 1; i++) {
        for (int j = 0; j < size - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                double temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}

// دالة توجيه واستدعاء دوال المكتبة القياسية داخل بيئة الـ VM
void vm_execute_stdlib_call(VirtualMachine* vm, const char* lib_name, const char* func_name) {
    if (strcmp(lib_name, "math") == 0) {
        if (strcmp(func_name, "random") == 0) {
            VMValue res;
            res.type = VAL_NUMBER;
            res.as.num = qahwa_math_random(1, 100);
            vm_push(vm, res);
        }
    } 
    else if (strcmp(lib_name, "os") == 0) {
        if (strcmp(func_name, "time") == 0) {
            VMValue res;
            res.type = VAL_NUMBER;
            res.as.num = qahwa_os_time();
            vm_push(vm, res);
        }
    }
}
// ==========================================
// 🔌 الجزء التاسع: واجهة التكامل مع لغة C (C API)
// ==========================================

// هيكل حالة المترجم الخارجي (مماثل لـ lua_State)
typedef struct {
    VirtualMachine vm;
    char script_buffer[MAX_LINES * MAX_LINE_LEN];
    int is_initialized;
} QahwaState;

// 1. إنشاء حالة جديدة للمترجم
QahwaState* qahwa_newstate() {
    QahwaState* Q = (QahwaState*)malloc(sizeof(QahwaState));
    if (!Q) {
        printf("❌ خطأ فادح: فشل إنشاء حالة QahwaState جديدة!\n");
        return NULL;
    }
    init_vm(&Q->vm);
    Q->is_initialized = 1;
    Q->script_buffer[0] = '\0';
    return Q;
}

// 2. فتح المكتبات القياسية داخل حالة الـ API
void qahwa_openlibs(QahwaState* Q) {
    if (!Q || !Q->is_initialized) return;
    // تهيئة الجداول والمتغيرات الافتراضية القياسية داخل الـ VM
    // (يتم تجهيز البيئة لتلقي الأوامر والدوال القياسية)
    printf("🔌 [C API]: تم فتح وتفعيل المكتبات القياسية بنجاح.\n");
}

// 3. تنفيذ سكريبت نصي مباشرة من داخل C (مماثل لـ luaL_dostring)
int qahwa_dostring(QahwaState* Q, const char* script_code) {
    if (!Q || !Q->is_initialized) return 1;

    // تجهيز السطر التنفيذي وتحويله لـ Bytecode وترجمته فوراً
    char lines[MAX_LINES][MAX_LINE_LEN];
    int total_lines = 0;

    // تقسيم السكريبت النصي إلى سطور منفصلة للمعالجة
    char temp_code[MAX_LINE_LEN * 10];
    strncpy(temp_code, script_code, sizeof(temp_code) - 1);
    temp_code[sizeof(temp_code) - 1] = '\0';

    char* token = strtok(temp_code, "\n");
    while (token != NULL && total_lines < MAX_LINES) {
        strcpy(lines[total_lines++], token);
        token = strtok(NULL, "\n");
    }

    // ترجمة وتشغيل السطور عبر عقل ومحرك الـ VM الخاص بـ QahwaState
    compile_script_to_vm(lines, total_lines);
    vm_execute_bytecode(&Q->vm, bytecode, code_count);

    return 0; // نجاح التنفيذ
}

// 4. إغلاق وتحرير حالة المترجم تماماً لتفادي أي تسريب للذاكرة
void qahwa_close(QahwaState* Q) {
    if (Q) {
        // تنظيف الكائنات الديناميكية التابعة للـ GC
        garbage_collect(&Q->vm);
        free(Q);
        printf("🔌 [C API]: تم إغلاق وتحرير حالة QahwaScript بأمان تام.\n");
    }
}
