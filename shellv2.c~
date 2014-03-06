#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <string.h>
#include <ctype.h>
#include <malloc.h>

#define MAX_CMD_LEN 256
#define TRUE 1
#define FALSE 0
#define EXIT -1

int tokenize(char **tokens, char* input, char *delimiter);
char *trim(char *str);
char *trimmedSubstr(char *src, int start, int len);
char **breakIntoFour(char *str);
void prompt();
void parse(char *str);
void pipeCommand(char **tokens, int tokenCount);
int inbuiltCommands(char *input);
void cdCommand(char *args);
void addToHistory(char *input);
void printHistory(void);
void printLastNHistory(int n);
char *getFromHistory(int num);
void basicCommand(char *str);

char *historyBuffer[MAX_CMD_LEN];
int curHistory=0;

int main(){
    while(TRUE) {
        prompt();
        char *command = (char*)malloc(sizeof(char)*256);
        fgets(command, MAX_CMD_LEN, stdin);
        if(!strcmp(command, "\n")) continue;    // If user just hit enter
        command = strtok(command, "\n");
        addToHistory(command);
        int is_inbuilt = inbuiltCommands(command);
        if(is_inbuilt==TRUE) continue;
        else if(is_inbuilt==EXIT){
            printf("%s\n", "Exiting Shell...");
            break;
        }
        else if(is_inbuilt==FALSE){
            parse(command);
        }
    }
    return 0;
}

void parse(char *str){
    char **tokens = (char **)malloc(sizeof(char *)*256);
    int tokenCount = tokenize(tokens, str, "|");

    if(tokenCount==1){
        basicCommand(tokens[0]);
    }
    else if(tokenCount>1){
        pipeCommand(tokens, tokenCount);
    }
}

void pipeCommand(char **tokens, int tokenCount){
    int numPipes = tokenCount-1;
    int fds[numPipes][2];
    int i;
    for(i=0; i<numPipes; i++){
        if((pipe(fds[i]))==-1){
            fprintf(stderr, "%s\n", "Pipe error in pipeCommand");
            return;
        }
    }

    char **args = (char **)malloc(sizeof(char *)*256);
    int argCount = tokenize(args, tokens[0], " ");
    args[argCount] = NULL;
    pid_t pid;
    if((pid=fork())==0){
        dup2(fds[0][1], 1);
        close(fds[0][0]);
        execvp(args[0], args);
    }
    close(fds[0][1]);
    waitpid(pid, NULL, 0);

    for(i=1; i<numPipes; i++){
        argCount = tokenize(args, tokens[i], " ");
        args[argCount] = NULL;
        if((pid=fork())==0){
            dup2(fds[i-1][0], 0);
            close(fds[i-1][1]);
            dup2(fds[i][1], 1);
            close(fds[i][0]);
            execvp(args[0], args);
        }
        close(fds[i-1][0]);
        close(fds[i-1][1]);
        close(fds[i][1]);
        waitpid(pid, NULL, 0);
    }

    argCount = tokenize(args, tokens[tokenCount-1], " ");
    args[argCount] = NULL;
    if((pid=fork())==0){
        dup2(fds[numPipes-1][0], 0);
        close(fds[numPipes-1][1]);
        execvp(args[0], args);
    }
    close(fds[numPipes-1][0]);
    close(fds[numPipes-1][1]);
    waitpid(pid, NULL, 0);

    for(i=0; i<numPipes; i++){
        close(fds[i][0]);
        close(fds[i][1]);
    }
}

void basicCommand(char *str){
    char **fourParts = breakIntoFour(str);
    char **args = (char **)malloc(sizeof(char *)*256);
    int argCount = tokenize(args, fourParts[0], " ");
    args[argCount] = NULL;
    // int wait_ = TRUE;
    // if(!strcmp(args[argCount-1], "&")){
    //     wait = FALSE;
    //     args[argCount-1] = NULL;
    // }
    if(fourParts[1]==NULL && fourParts[2]==NULL && fourParts[3]==NULL){
        pid_t pid;
        if((pid=fork())==0){
            execvp(args[0], args);
        }
        else {
            wait(NULL);
        }
    }
    else if(fourParts[1]!=NULL && fourParts[2]==NULL && fourParts[3]==NULL){
        int fds[2];
        pipe(fds);
        pid_t child1;
        pid_t child2;
        if((child1=fork())==0){
            dup2(fds[0], 0);
            close(fds[1]);
            execvp(args[0],args);
        }
        else if((child2=fork())==0){
            int fd = open(fourParts[1], O_RDWR , 0777);
            char c;
            while (read(fd, &c, 1) > 0) write(fds[1], &c, 1);
            close(fds[0]);
        }
        else {
            waitpid(child1, NULL, 0);
        }
    }
    else if(fourParts[1]==NULL && (fourParts[2]!=NULL || fourParts[3]!=NULL)){
        int fds[2];
        pipe(fds);

        pid_t child;

        int fd;
        if(fourParts[2]!=NULL) fd = open(fourParts[2], O_WRONLY | O_CREAT, 0666);
        else if(fourParts[3]!=NULL) fd = open(fourParts[3], O_RDWR | O_APPEND | O_CREAT, 0777);

        char c;
        if((child=fork())==0){
            dup2(fds[1], 1);
            close(fds[0]);
            execvp(args[0],args);
        }
        close(fds[1]);
        while (read(fds[0], &c, 1) > 0) write(fd, &c, 1);
        waitpid(child, NULL, 0);
    }
    else if(fourParts[1]!=NULL && (fourParts[2]!=NULL || fourParts[3]!=NULL)){
        int fdRead[2];
        pipe(fdRead);
        int fdWrite[2];
        pipe(fdWrite);

        pid_t child;
        int fd1 = open(fourParts[1], O_RDWR , 0777);
        char c;
        while (read(fd1, &c, 1) > 0) write(fdRead[1], &c, 1);
        if((child=fork())==0){
            dup2(fdRead[0], 0);
            close(fdRead[1]);
            dup2(fdWrite[1], 1);
            close(fdWrite[0]);
            execvp(args[0],args);
        }
        close(fdRead[0]);
        close(fdRead[1]);
        close(fdWrite[1]);
        char d;
        int fd2;
        if(fourParts[2]!=NULL) fd2 = open(fourParts[2], O_WRONLY | O_CREAT, 0666);
        else if(fourParts[3]!=NULL) fd2 = open(fourParts[3], O_RDWR | O_APPEND | O_CREAT, 0777);
        while(read(fdWrite[0], &d, 1)>0) write(fd2, &d, 1);
        waitpid(child, NULL, 0);

    }

}



int tokenize(char **tokens, char* input, char *delimiter) {
    // making a copy of input string
    int len = strlen(input);
    char *str = (char *)malloc(sizeof(char)*len+1);
    strncpy(str, input, len+1);
    // printf("%s\n", str);

    // char **tokens = (char *)malloc(sizeof(char *)*256);

    char *tok = (char *)malloc(sizeof(char)*256);
    tok = strtok(str, delimiter);
    int i = 0;
    while(tok != NULL) {
        tokens[i] = trim(tok);
        tok = (char *)malloc(sizeof(char)*256);
        tok = strtok(NULL, delimiter);
        i++;
    }
    return i;   // returns the array of tokens
}

char *trim(char *str) {
    int len = strlen(str);
    char *tmp = (char *)malloc(sizeof(char)*len+1);
    strncpy(tmp, str, len);
    tmp[len] = '\0';

    int i;
    for(i = 0; i < len; i++){
        if(str[i]!=' ') break;
    }
    int start = i;
    for(i = len-1; i >= 0; i--){
        if(str[i]!=' ') break;
    }
    int end = i+1;
    int copyLen = end-start;
    int j = 0;
    for(i=start; i<end; i++){
        tmp[j] = str[i];
        j++;
    }
    tmp[j] = '\0';
    return tmp;
}

char *trimmedSubstr(char *src, int start, int len){
    char *dest = (char *)malloc(sizeof(char)*64);
    int i, j=0;
    for(i=start; i<start+len; i++){
        dest[j]=src[i];
        j++;
    }
    dest[len] = '\0';
    return trim(dest);
}

char **breakIntoFour(char *str){
    int i;
    int len = strlen(str);
    int writeToStart=-1, writeToLen=-1, writeToMode=-1;
    int readFromStart=-1, readFromLen=-1;
    int cmdStart=0, cmdLen=-1;
    for(i=0; i<len; i++){
        if(str[i]=='>'){
            if(str[i+1]=='>'){
                writeToStart = i+2;
                writeToMode=1;          // Append
                i++;
            }
            else{
                writeToStart = i+1;
                writeToMode=0;          // Write
            }
            writeToLen = len-writeToStart;
        }
        else if(str[i]=='<'){
            readFromStart = i+1;
        }
    }
    if(readFromStart>0){
        if(writeToMode<0){
            readFromLen = len-readFromStart;
        }
        else if(writeToMode==0){
            readFromLen = writeToStart -1 -readFromStart;
        }
        else if(writeToMode==1){
            readFromLen = writeToStart -2 -readFromStart;
        }
        cmdLen = readFromStart-1;
    }
    else if(writeToMode==0) cmdLen = writeToStart-1;
    else if(writeToMode==1) cmdLen = writeToStart-2;
    else cmdLen = len;

    char **fourParts = (char **)malloc(sizeof(char *)*4);
    for(i=0; i<4; i++) fourParts[i]=(char*)malloc(sizeof(char)*32);

    fourParts[0] = trimmedSubstr(str, cmdStart, cmdLen);    // extracting cmd

    if(readFromStart<0) fourParts[1]=NULL;                  //extracting readFrom
    else fourParts[1]=trimmedSubstr(str, readFromStart, readFromLen);

    fourParts[2]=NULL; fourParts[3]=NULL;
    if(writeToMode==0) fourParts[2]=trimmedSubstr(str, writeToStart, writeToLen);
    else if(writeToMode==1) fourParts[3]=trimmedSubstr(str, writeToStart, writeToLen);

    return fourParts;

}

void prompt(){
    char cwd[100];
    getcwd(cwd, 100);
    printf("MyShell@%s>>>", cwd);
}

int inbuiltCommands(char *input)
{
    int mode=0;         /*mode = 1 for cd, history,! ; 0 for inbuilt commands; -1 for exit*/
    char ** args=(char **)malloc(sizeof(char *)*256);
    int tokenCount=tokenize(args,input," ");
    //printf("%s,%s,%s\n",input,args[0],args[1]);
    if(!strcmp(args[0], "cd"))
    {
        mode=1;
        //printf("Here");
        cdCommand(args[1]);
        char cwd[100];
        getcwd(cwd,100);
        // printf("%s\n",cwd);
    }

    else if(!(strcmp(args[0],"history"))){
        if(tokenCount==2){
            //printf("here\n");
            int n = atoi(args[1]);
            if(n<=0){
                fprintf(stderr, "%s\n", "Invalid argument. Argument must be a non negative number");
            }

            else printLastNHistory(n);
        }
        else{
            printf("here\n");
            printHistory();
        }

        mode=1;


    }

    else if(!(strcmp(args[0],"exit"))){
        //printf("mm\n");
        return -1;
    }


    else if(!strcmp(args[0],"!")){

        mode=1;
        char * historyAtnumber = (char *)malloc(sizeof(char) *256);
        historyAtnumber=getFromHistory(atoi(args[1]));
        int ret = inbuiltCommands(historyAtnumber);
        if(ret <= 0){
        	parse(historyAtnumber);
        }
    }

    else if(!(strcmp(args[0],"echo"))){
        mode = 1;
    ;
    //printf("outside tokencount==2\n");
        if(tokenCount==2){
            printf("%s\n",args[1]);
            //printf("%s\n",args[1]);
        }
    }

    // else{
    //     printf("in else\n");
    //     mode=0;

    //     parse(input);
    // }

return mode;

}

void addToHistory(char *input)
{
    char *arr= malloc(sizeof(char)*MAX_CMD_LEN);
    int j;

    for(j=0;j<strlen(input);j++)
        {
        arr[j]=input[j];
        }
    arr[j] = '\0';
    historyBuffer[curHistory++] = arr;


}

void printHistory(void)
{
    int c=1,ai;

    for(ai=0;ai<curHistory;ai++)
    {
        printf("%d    %s\n",c,historyBuffer[ai]);
        c++;
    }
}

void printLastNHistory(int n){
    int c=curHistory-n,ai;
    if(c<0){
        c=0;
    }
    for(ai=c;ai<curHistory;ai++)
    {
        printf("%d    %s\n",ai+1,historyBuffer[ai]);
        c++;
    }
}

char *getFromHistory(int num)
{
    printf("%s\n",historyBuffer[num-1]);
    return historyBuffer[num-1];
}

void cdCommand(char *args)
{
    int ret;
     //printf("%s",args[1]);
    char cmndbuf[MAX_CMD_LEN];
    if (!*(args+1))
        strcpy(cmndbuf, "pwd");
    if((ret = chdir(args))<0){
        fprintf(stderr, "%s\n", "No such directory. Please check your input.");
    }
    strcpy(cmndbuf, "pwd");
}
