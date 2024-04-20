#include <iostream>

using namespace std;

int MAX_KEYS = 3;

struct BTreeNode {
    int keys[MAX_KEYS];
    BTreeNode* children[MAX_KEYS + 1];
    int numKeys;
    bool isLeaf;

    BTreeNode() {
        numKeys = 0;
        isLeaf = true;
        for (int i = 0; i < MAX_KEYS + 1; ++i)
            children[i] = nullptr;
    }
};

class BTree {
private:
    BTreeNode* root;
    int minDegree;

    void splitChild(BTreeNode* parent, int index);
    void insertNonFull(BTreeNode* node, int key);
    bool searchInNode(BTreeNode* node, int key, int& position);
    void remove(BTreeNode* node, int key);
    void removeFromLeaf(BTreeNode* node, int index);
    void removeFromNonLeaf(BTreeNode* node, int index);
    int getPredecessor(BTreeNode* node, int index);
    int getSuccessor(BTreeNode* node, int index);
    void fill(BTreeNode* node, int index);
    void borrowFromPrev(BTreeNode* node, int index);
    void borrowFromNext(BTreeNode* node, int index);
    void merge(BTreeNode* node, int index);
    void display(BTreeNode* node, int level);

public:
    BTree(int _minDegree);
    ~BTree();

    void insert(int key);
    bool search(int key);
    void remove(int key);
    void display(); // Corrected declaration
    void menu();
};

BTree::BTree(int _minDegree) {
    root = nullptr;
    minDegree = _minDegree;
}

BTree::~BTree() {}

void BTree::splitChild(BTreeNode* parent, int index) {
    BTreeNode* child = parent->children[index];
    BTreeNode* newChild = new BTreeNode();
    newChild->isLeaf = child->isLeaf;
    newChild->numKeys = minDegree - 1;
    for (int i = 0; i < minDegree - 1; ++i)
        newChild->keys[i] = child->keys[i + minDegree];
    if (!child->isLeaf) {
        for (int i = 0; i < minDegree; ++i)
            newChild->children[i] = child->children[i + minDegree];
    }
    child->numKeys = minDegree - 1;
    for (int i = parent->numKeys; i >= index + 1; --i)
        parent->children[i + 1] = parent->children[i];
    parent->children[index + 1] = newChild;
    for (int i = parent->numKeys - 1; i >= index; --i)
        parent->keys[i + 1] = parent->keys[i];
    parent->keys[index] = child->keys[minDegree - 1];
    parent->numKeys++;
}

void BTree::insertNonFull(BTreeNode* node, int key) {
    int i = node->numKeys - 1;
    if (node->isLeaf) {
        while (i >= 0 && key < node->keys[i]) {
            node->keys[i + 1] = node->keys[i];
            i--;
        }
        node->keys[i + 1] = key;
        node->numKeys++;
    } else {
        while (i >= 0 && key < node->keys[i])
            i--;
        i++;
        if (node->children[i]->numKeys == 2 * minDegree - 1) {
            splitChild(node, i);
            if (key > node->keys[i])
                i++;
        }
        insertNonFull(node->children[i], key);
    }
}

void BTree::insert(int key) {
    if (root == nullptr) {
        root = new BTreeNode();
        root->keys[0] = key;
        root->numKeys = 1;
    } else {
        if (root->numKeys == 2 * minDegree - 1) {
            BTreeNode* newRoot = new BTreeNode();
            newRoot->isLeaf = false;
            newRoot->children[0] = root;
            splitChild(newRoot, 0);
            root = newRoot;
        }
        insertNonFull(root, key);
    }
}

bool BTree::search(int key) {
    int position = -1;
    bool found = searchInNode(root, key, position);
    if (found)
        cout << key << " found at position " << position << endl;
    else
        cout << key << " not found in the B-tree." << endl;
    return found;
}

bool BTree::searchInNode(BTreeNode* node, int key, int& position) {
    int i = 0;
    while (i < node->numKeys && key > node->keys[i])
        i++;
    if (i < node->numKeys && key == node->keys[i]) {
        position = i;
        return true;
    }
    if (node->isLeaf)
        return false;
    return searchInNode(node->children[i], key, position);
}

void BTree::remove(int key) {
    if (!root) {
        cout << "The tree is empty." << endl;
        return;
    }

    remove(root, key);
    if (root->numKeys == 0) {
        BTreeNode* tmp = root;
        if (root->isLeaf)
            root = nullptr;
        else
            root = root->children[0];
        delete tmp;
    }
}

void BTree::remove(BTreeNode* node, int key) {
    int index = 0;
    while (index < node->numKeys && key > node->keys[index])
        ++index;

    if (index < node->numKeys && key == node->keys[index]) {
        if (node->isLeaf)
            removeFromLeaf(node, index);
        else
            removeFromNonLeaf(node, index);
    } else {
        if (node->isLeaf) {
            cout << "The key " << key << " is not present in the tree." << endl;
            return;
        }

        bool flag = (index == node->numKeys);

        if (node->children[index]->numKeys < minDegree)
            fill(node, index);

        if (flag && index > node->numKeys)
            remove(node->children[index - 1], key);
        else
            remove(node->children[index], key);
    }
}

void BTree::removeFromLeaf(BTreeNode* node, int index) {
    for (int i = index + 1; i < node->numKeys; ++i)
        node->keys[i - 1] = node->keys[i];
    node->numKeys--;
}

void BTree::removeFromNonLeaf(BTreeNode* node, int index) {
    int key = node->keys[index];

    if (node->children[index]->numKeys >= minDegree) {
        int pred = getPredecessor(node, index);
        node->keys[index] = pred;
        remove(node->children[index], pred);
    } else if (node->children[index + 1]->numKeys >= minDegree) {
        int succ = getSuccessor(node, index);
        node->keys[index] = succ;
        remove(node->children[index + 1], succ);
    } else {
        merge(node, index);
        remove(node->children[index], key);
    }
}

int BTree::getPredecessor(BTreeNode* node, int index) {
    BTreeNode* curr = node->children[index];
    while (!curr->isLeaf)
        curr = curr->children[curr->numKeys];
    return curr->keys[curr->numKeys - 1];
}

int BTree::getSuccessor(BTreeNode* node, int index) {
    BTreeNode* curr = node->children[index + 1];
    while (!curr->isLeaf)
        curr = curr->children[0];
    return curr->keys[0];
}

void BTree::fill(BTreeNode* node, int index) {
    if (index != 0 && node->children[index - 1]->numKeys >= minDegree)
        borrowFromPrev(node, index);
    else if (index != node->numKeys && node->children[index + 1]->numKeys >= minDegree)
        borrowFromNext(node, index);
    else {
        if (index != node->numKeys)
            merge(node, index);
        else
            merge(node, index - 1);
    }
}

void BTree::borrowFromPrev(BTreeNode* node, int index) {
    BTreeNode* child = node->children[index];
    BTreeNode* sibling = node->children[index - 1];

    for (int i = child->numKeys - 1; i >= 0; --i)
        child->keys[i + 1] = child->keys[i];

    if (!child->isLeaf) {
        for (int i = child->numKeys; i >= 0; --i)
            child->children[i + 1] = child->children[i];
    }

    child->keys[0] = node->keys[index - 1];

    if (!child->isLeaf)
        child->children[0] = sibling->children[sibling->numKeys];

    node->keys[index - 1] = sibling->keys[sibling->numKeys - 1];

    child->numKeys++;
    sibling->numKeys--;
}

void BTree::borrowFromNext(BTreeNode* node, int index) {
    BTreeNode* child = node->children[index];
    BTreeNode* sibling = node->children[index + 1];

    child->keys[child->numKeys] = node->keys[index];

    if (!child->isLeaf)
        child->children[child->numKeys + 1] = sibling->children[0];

    node->keys[index] = sibling->keys[0];

    for (int i = 1; i < sibling->numKeys; ++i)
        sibling->keys[i - 1] = sibling->keys[i];

    if (!sibling->isLeaf) {
        for (int i = 1; i <= sibling->numKeys; ++i)
            sibling->children[i - 1] = sibling->children[i];
    }

    child->numKeys++;
    sibling->numKeys--;
}

void BTree::merge(BTreeNode* node, int index) {
    BTreeNode* child = node->children[index];
    BTreeNode* sibling = node->children[index + 1];

    child->keys[minDegree - 1] = node->keys[index];

    for (int i = 0; i < sibling->numKeys; ++i)
        child->keys[i + minDegree] = sibling->keys[i];

    if (!child->isLeaf) {
        for (int i = 0; i <= sibling->numKeys; ++i)
            child->children[i + minDegree] = sibling->children[i];
    }

    for (int i = index + 1; i < node->numKeys; ++i)
        node->keys[i - 1] = node->keys[i];

    for (int i = index + 2; i <= node->numKeys; ++i)
        node->children[i - 1] = node->children[i];

    child->numKeys += sibling->numKeys + 1;
    node->numKeys--;

    delete sibling;
}

void BTree::display() {
    display(root, 0);
}

void BTree::display(BTreeNode* node, int level) {
    if (node != nullptr) {
        for (int i = node->numKeys - 1; i >= 0; --i) {
            display(node->children[i + 1], level + 1);
            for (int j = 0; j < level; ++j)
                cout << "    ";
            cout << node->keys[i] << endl;
        }
        display(node->children[0], level + 1);
    }
}

void BTree::menu() {
    int choice, key;
    do {
        cout << "\nB-Tree Operations\n";
        cout << "1. Insert\n";
        cout << "2. Search\n";
        cout << "3. Delete\n";
        cout << "4. Display\n";
        cout << "5. Exit\n";
        cout << "Enter your choice: ";
        cin >> choice;

        switch (choice) {
            case 1:
                cout << "Enter key to insert: ";
                cin >> key;
                insert(key);
                break;
            case 2:
                cout << "Enter key to search: ";
                cin >> key;
                search(key);
                break;
            case 3:
                cout << "Enter key to delete: ";
                cin >> key;
                remove(key);
                break;
            case 4:
                cout << "B-tree:\n";
                display();
                break;
            case 5:
                cout << "Exiting...\n";
                break;
            default:
                cout << "Invalid choice. Please try again.\n";
        }
    } while (choice != 5);
}

int main() {
    BTree btree(2);

    btree.menu();
  
    return 0;
}

