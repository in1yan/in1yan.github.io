+++
title = "Linked List"
date = "2025-03-17T05:03:37Z"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "iniyan"
authorTwitter = "" #do not include @
cover = ""
tags = ["code", "ds", "linked list"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = true
hideComments = false
+++

Here I am 👋, it's been 5 months or so since i wrote my first blog and this is my second.
Yeah that's my level of productivity 😶‍🌫️.
In my second semester I have **data structures** which to me it is hard to grasp, at least these linked lists. So, in this blog I'm going deep into the linked lists (not so deep).
<!--more-->
![deep thoughts with deep](https://media1.tenor.com/m/RBVr8JLidQkAAAAd/the-deep-deep-thoughts-with-the-deep.gif)
# What are these linked list?

Linked lists are similar to array which let's you store data in a sequence . Unlike array in a linked list the data can be anywhere but they will be linked . That's the catch. In linked list each **node** will point to the next node(`singly linked list`) and the last node will point to `NULL` or nothing.

![pointers](https://cdn-useast1.kapwing.com/static/templates/3-spiderman-pointing-meme-template-regular-237687a9.webp)

# Why not just use array?

 Arrays are cool and all, but they have their limitations. The main issue? **Fixed size.** When you create an array, you have to define its size beforehand. If you need more space later .You either create a new, bigger array and copy everything over (which is slow) or just deal with running out of space.

Linked lists, on the other hand, are dynamic. Need more space? Just create a new node and link it. No need to worry about resizing or shifting elements around.

Another issue is **insertion and deletion.** In arrays, inserting a new element in the middle means shifting everything after it to the right. Deleting an element means shifting everything back to fill the gap. That’s slow. In a linked list, you just change a couple of pointers, and you’re done. No shifting required.

But before you start thinking linked lists are the ultimate data structure, let’s talk about the downsides.

# The Dark Side of Linked Lists
![deep thoughts with deep](https://media2.giphy.com/media/v1.Y2lkPTc5MGI3NjExdnJoNGdsaW4weDBiY2F6dmp4ZXhhMnczaWFxeWV1cnQxbjFtdnU4eiZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/Txtqd7gaRciwDogqqC/giphy.gif)
Everything has a price, and for linked lists, that price is **extra memory and slower access.**

1. **Memory Overhead** – Every node in a linked list stores not just the data but also a pointer (or two, if it’s a doubly linked list). That means more memory usage compared to arrays.
    
2. **Slow Access** – In an array, accessing the 10th element is instant (`O(1)`). Just `arr[9]`, and boom, you got it. In a linked list? You start at the first node and follow the pointers one by one until you reach the 10th node. That’s `O(n)`. Not ideal if you need fast lookups.
    

So yeah, **linked lists are powerful, but not always the best choice.**

# Types of Linked Lists

1. **Singly Linked List** – Each node points to the next one. The last node points to `NULL`. Simple and common.
    
2. **Doubly Linked List** – Each node has two pointers: one to the next node and one to the previous node. This makes traversal easier (you can go forward and backward), but it takes up more memory.
    
3. **Circular Linked List** – The last node points back to the first node instead of `NULL`. Can be singly or doubly linked.
    

# Implementing a Linked List in C

Alright, enough theory. Let’s get our hands dirty with some code. Here’s a basic implementation of a singly linked list in C:

{{<code language="c" title="Implementation" open="true" linenos="true">}}
#include <stdio.h>
#include <stdlib.h>

// Node structure
typedef struct Node {
  int data;
  struct Node* next;
} Node;

// Function to create a new node
Node* createNode(int data) {
  Node* newNode = (Node*)malloc(sizeof(Node));
  newNode->data = data;
  newNode->next = NULL;
  return newNode;
}

// Function to print the linked list
void printList(Node* head) {
  Node* temp = head;
  while (temp != NULL) {
    printf("%d -> ", temp->data);
    temp = temp->next;
  }
  printf("NULL\n");
}

// Function to insert a node at the end
void insertEnd(Node** head, int data) {
  Node* newNode = createNode(data);
  if (*head == NULL) {
    *head = newNode;
    return;
  }
  Node* temp = *head;
  while (temp->next != NULL) {
    temp = temp->next;
  }
  temp->next = newNode;
}

// Function to delete a node
void deleteNode(Node** head, int key) {
  Node* temp = *head, *prev = NULL;

  if (temp != NULL && temp->data == key) {
    *head = temp->next;
    free(temp);
    return;
  }

  while (temp != NULL && temp->data != key) {
    prev = temp;
    temp = temp->next;
  }

  if (temp == NULL) return;

  prev->next = temp->next;
  free(temp);
}

// Main function
int main() {
  Node* head = NULL;
  insertEnd(&head, 10);
  insertEnd(&head, 20);
  insertEnd(&head, 30);

  printf("Linked List: ");
  printList(head);

  deleteNode(&head, 20);
  printf("After deleting 20: ");
  printList(head);

  return 0;
}
{{</code>}}


That’s it for now. If you made it this far, Thank you.
