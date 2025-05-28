#include <iostream>
#include <string>
#include <vector>
#include <iomanip>
#include <limits>
#include <fstream>
#include <sstream>
#include <chrono>
#include <map>
#include <functional>
#include <cstdlib>
#include <random>
#include <algorithm>
#include <ctime>
#include <regex>

using namespace std;

// Constants for file storage and interest calculation
const string ACCOUNTS_FILE = "bank_accounts.txt";
const double INTEREST_RATE = 0.03;
const int INTEREST_MONTH_1 = 6;
const int INTEREST_DAY_1 = 30;
const int INTEREST_MONTH_2 = 12;
const int INTEREST_DAY_2 = 31;

// Transaction class: Represents a single transaction for a bank account
class Transaction {
public:
    string id;
    string type;
    double amount;
    string datetime;
    string otherAccount;

    // Constructor for creating a new transaction
    Transaction(const string& type, double amount, const string& otherAccount = "") : type(type), amount(amount), otherAccount(otherAccount) {
        stringstream ss;
        ss << time(nullptr) << "_" << rand();
        id = ss.str();

        time_t now = time(nullptr);
        tm* now_tm = localtime(&now);
        stringstream dt;
        dt << put_time(now_tm, "%Y-%m-%d %H:%M:%S");
        datetime = dt.str();
    }

    // Constructor for loading a transaction from file
    Transaction(const string& id, const string& type, double amount, const string& datetime, const string& otherAccount)
        : id(id), type(type), amount(amount), datetime(datetime), otherAccount(otherAccount) {}
};

// BankAccount class: Represents a user's bank account and its operations
class BankAccount {
private:
    string accountNumber;
    string username;
    string passwordHash;
    double balance;
    string email;
    string nameSurname;
    time_t lastInterestApplied = 0;
    vector<Transaction> transactions;

public:
    // Constructor for creating or loading an account
    BankAccount(string number, string username, const string& pwdHash, double initialBalance, const string& emailAddr, const string& nameSurname, time_t lastInterest = 0)
        : accountNumber(number), username(username), passwordHash(pwdHash), balance(initialBalance), email(emailAddr), nameSurname(nameSurname), lastInterestApplied(lastInterest) {}

    // Deposit a specified amount into the account
    void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
            cout << "Deposit successful: " << fixed << setprecision(2) << amount
                 << " $. New balance: " << balance << " $" << endl;
            addTransaction(Transaction("DEPOSIT", amount));
        } else {
            cout << "Invalid deposit amount" << endl;
        }
    }

    // Withdraw a specified amount from the account
    void withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            cout << "Withdrawal successful: " << fixed << setprecision(2) << amount
                 << " $. New balance: " << balance << " $" << endl;
            addTransaction(Transaction("WITHDRAWAL", amount));
        } else {
            cout << "Invalid withdrawal amount or insufficient funds" << endl;
        }
    }

    // Transfer a specified amount to another account
    bool transfer(BankAccount &recipient, double amount) {
        if (amount > 0 && balance >= amount) {
            balance -= amount;
            recipient.balance += amount;
            cout << "Transfer successful: " << fixed << setprecision(2) << amount
                 << " $ to account " << recipient.accountNumber << endl;
            addTransaction(Transaction("TRANSFER_SENT", amount, recipient.accountNumber));
            recipient.addTransaction(Transaction("TRANSFER_RECEIVED", amount, accountNumber));
            return true;
        } else {
            cout << "Transfer failed. Insufficient funds or invalid amount" << endl;
            return false;
        }
    }

    // Add a transaction to the account's transaction history
    void addTransaction(const Transaction& t) {
        transactions.push_back(t);
        saveTransactionsToFile();
    }

    // Get all transactions for this account
    const vector<Transaction>& getTransactions() const { return transactions; }

    // Save all transactions to a file for this account
    void saveTransactionsToFile() const {
        string filename = accountNumber + "_transactions.txt";
        ofstream out(filename);
        for (const auto& t : transactions) {
            out << t.id << "," << t.type << "," << fixed << setprecision(2) << t.amount << "," << t.datetime << "," << t.otherAccount << "\n";
        }
    }

    // Load all transactions from the account's transaction file
    void loadTransactionsFromFile() {
        transactions.clear();
        string filename = accountNumber + "_transactions.txt";
        ifstream in(filename);
        string line;
        while (getline(in, line)) {
            stringstream ss(line);
            string id, type, amountStr, datetime, otherAccount;
            getline(ss, id, ',');
            getline(ss, type, ',');
            getline(ss, amountStr, ',');
            getline(ss, datetime, ',');
            getline(ss, otherAccount);
            try {
                double amount = stod(amountStr);
                transactions.emplace_back(id, type, amount, datetime, otherAccount);
            } catch (...) {}
        }
    }

    // Display account information to the user
    void display() const {
        cout << "Number: " << accountNumber << endl;
        cout << "Username: " << username << endl;
        cout << "Name & Surname: " << nameSurname << endl;
        cout << "Email: " << email << endl;
        cout << "Balance: " << fixed << setprecision(2) << balance << " $" << endl;
    }

    // Getters and setters for account fields
    string getEmail() const { return email; }
    void setEmail(const string& newEmail) { email = newEmail; }
    string getAccountNumber() const { return accountNumber; }
    string getUsername() const { return username; }
    string getNameSurname() const { return nameSurname; }
    string getPasswordHash() const { return passwordHash; }
    double getBalance() const { return balance; }
    time_t getLastInterestApplied() const { return lastInterestApplied; }
    void setBalance(double newBalance) { balance = newBalance; }
    void setUsername(const string& uname) { username = uname; }
    void setNameSurname(const string& ns) { nameSurname = ns; }
    void setPasswordHash(const string& hash) { passwordHash = hash; }
    void setLastInterestApplied(time_t t) { lastInterestApplied = t; }
};

// Hash a password using std::hash for basic security
string hashPassword(const string& password) {
    hash<string> hasher;
    return to_string(hasher(password));
}

// Generate a random 6-digit OTP for email verification
string generateOTP() {
    unsigned seed = chrono::system_clock::now().time_since_epoch().count();
    mt19937 gen(seed);
    uniform_int_distribution<> dis(100000, 999999);
    return to_string(dis(gen));
}

// Send an OTP to the user's email using an external Python script
void sendOTPEmail(const string& email, const string& otp, const string& nameSurname = "") {
    string command = "python send_otp_email.py \"" + email + "\" \"" + otp + "\" \"" + nameSurname + "\"";
    system(command.c_str());
}

// Send interest information email after account creation
void sendInterestInfoEmail(const string& email, const string& nameSurname = "") {
    string command = "python send_interest_info_email.py \"" + email + "\" \"" + nameSurname + "\"";
    system(command.c_str());
}

// Check if a password is strong based on length and character requirements
bool isStrongPassword(const string& pwd) {
    if (pwd.length() < 12 || pwd.length() > 16) return false;
    bool hasUpper = false, hasLower = false, hasDigit = false, hasSpecial = false;
    string specials = "!@#$%^&*";
    for (char c : pwd) {
        if (isupper(c)) hasUpper = true;
        else if (islower(c)) hasLower = true;
        else if (isdigit(c)) hasDigit = true;
        else if (specials.find(c) != string::npos) hasSpecial = true;
    }
    return hasUpper && hasLower && hasDigit && hasSpecial;
}

// Check if a username is valid based on format rules
bool isValidUsername(const string& username) {
    if (username.length() < 5 || username.length() > 20) return false;
    regex pattern("^[a-zA-Z0-9_]+$");
    if (!regex_match(username, pattern)) return false;
    if (username.front() == '_' || username.back() == '_') return false;
    if (username.find("__") != string::npos) return false;
    return true;
}

// BankSystem class: Manages all accounts, user sessions, and main banking operations
class BankSystem {
private:
    vector<BankAccount> accounts;
    BankAccount* currentAccount = nullptr;
    map<string, pair<int, chrono::system_clock::time_point>> loginAttemptsMap;
    const int MAX_ATTEMPTS = 3;
    const int LOCK_DURATION_SECONDS = 24 * 60 * 60;

    // Find an account by account number
    BankAccount* findAccount(const string& accountNumber) {
        for (auto& account : accounts) {
            if (account.getAccountNumber() == accountNumber) {
                return &account;
            }
        }
        return nullptr;
    }

public:
    // Load all accounts from file when BankSystem is created
    BankSystem() {
        loadAccountsFromFile();
    }

    // Save all accounts to file when BankSystem is destroyed
    ~BankSystem() {
        saveAccountsToFile();
    }

    // Attempt to log in using account number and password
    bool login(const string& accountNumber, const string& password) {
        BankAccount* account = findAccount(accountNumber);
        if (account && account->getPasswordHash() == hashPassword(password)) {
            currentAccount = account;
            return true;
        }
        return false;
    }

    // Log out the current user
    void logout() {
        currentAccount = nullptr;
        cout << "Logged out successfully." << endl;
    }

    // Generate a unique account number with a given prefix and length
    string generateUniqueAccountNumber(const vector<BankAccount>& accounts, int length, const string& prefix) {
        random_device rd;
        mt19937 gen(rd());
        uniform_int_distribution<> dis(0, 9);

        string accNumber;
        bool unique = false;
        while (!unique) {
            accNumber = prefix;
            while (accNumber.length() < length) {
                accNumber += to_string(dis(gen));
            }
            unique = true;
            for (const auto& acc : accounts) {
                if (acc.getAccountNumber() == accNumber) {
                    unique = false;
                    break;
                }
            }
        }
        return accNumber;
    }

    // Create a new user account with all required checks and OTP verification
    void createAccount() {
        string accNumber, username, pwd, email, nameSurname;
        double initialBalance;

        // Username input and validation
        while (true) {
            cout << "Username (5-20 chars, latin letters, digits, _, not start/end with _, no __, or press 0 to go back):" << endl;
            cout << "Note: This username will be permanent and cannot be changed later. Choose wisely! \n> ";
            cin >> username;
            if (username == "0") {
                cout << "Returning to main menu." << endl;
                return;
            }
            if (!isValidUsername(username)) {
                cout << "Invalid username! Please follow the username rules." << endl;
                continue;
            }
            if (usernameExists(username)) {
                cout << "Error: This username is already used by another account! Please try again..." << endl;
                continue;
            }
            break;
        }

        // Full name input and validation
        while (true) {
            cout << "Account Holder Name (Name & Surname, or press 0 to go back): ";
            cin.ignore();
            getline(cin, nameSurname);
            if (nameSurname == "0") {
                cout << "Returning to main menu." << endl;
                return;
            }
            if (nameSurname.find(' ') == string::npos) {
                cout << "Please enter both name and surname (e.g. John Smith)." << endl;
                continue;
            }
            break;
        }

        // Email input and OTP verification
        while (true) {
            cout << "Email (or press 0 to go back): ";
            cin >> email;
            if (email == "0") {
                cout << "Returning to menu." << endl;
                return;
            }
            bool emailUsed = false;
            for (const auto& acc : accounts) {
                if (acc.getEmail() == email) {
                    emailUsed = true;
                    break;
                }
            }
            if (emailUsed) {
                cout << "The email you entered is either already associated with another bank account or it doesn't exist. Please try again." << endl;
                continue;
            }
            int otpResult = verifyEmailOTP(email, nameSurname);
            if (otpResult == 1) {
                break;
            } else if (otpResult == -3) {
                cout << "The email you entered does not exist or is invalid. Please try again." << endl;
                continue;
            } else if (otpResult == -2) {
                cout << "OTP expired." << endl;
                cout << "Please check that your email address is correct and try again!" << endl;
            } else if (otpResult == -1) {
                cout << "Returning to menu." << endl;
                return;
            } else {
                cout << "Please check that your email address is correct and try again!" << endl;
            }
        }

        // Password input and validation
        do {
            cout << "Password (12-16 chars, 1 uppercase, 1 lowercase, 1 digit, 1 special !@#$%^&*): ";
            cin >> pwd;
            if (pwd == "0") {
                cout << "Returning to main menu." << endl;
                return;
            }
            if (!isStrongPassword(pwd)) {
                cout << "Weak password! Please follow the password rules." << endl;
                continue;
            }
            cout << "Password successful!" << endl;
            break;
        } while (true);
        string pwdHash = hashPassword(pwd);

        // Initial deposit input
        cout << "Initial Balance (or press 0 to go back): ";
        cin >> initialBalance;
        if (initialBalance == 0) {
            cout << "Returning to main menu." << endl;
            return;
        }

        // Generate and display new account number
        accNumber = generateUniqueAccountNumber(accounts, 12, "51");
        cout << "Your new Unique Account Number is: " << accNumber << endl;

        // Create and save the new account
        accounts.emplace_back(accNumber, username, pwdHash, initialBalance, email, nameSurname);
        accounts.back().addTransaction(Transaction("ACCOUNT_CREATED", initialBalance));
        saveAccountsToFile();
        cout << "Account created successfully!" << endl;

        // Send interest info email to the new user
        sendInterestInfoEmail(email, nameSurname);
        cout << "Press 0 to go back: ";
        string input;
        do {
            cin >> input;
        } while (input != "0");
        cout << "Returning to main menu." << endl;
    }

    // Deposit funds for the currently logged-in user
    void deposit() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        double amount;
        cout << "Amount to deposit (or press 0 to go back): ";
        cin >> amount;
        if (amount == 0) {
            cout << "Returning to menu." << endl;
            return;
        }
        currentAccount->deposit(amount);
        saveAccountsToFile();
        cout << "Press 0 to go back: ";
        string input;
        cin >> input;
    }

    // Withdraw funds for the currently logged-in user
    void withdraw() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        double amount;
        cout << "Amount to withdraw (or press 0 to go back): ";
        cin >> amount;
        if (amount == 0) {
            cout << "Returning to menu." << endl;
            return;
        }
        currentAccount->withdraw(amount);
        saveAccountsToFile();
        cout << "Press 0 to go back: ";
        string input;
        cin >> input;
    }

    // Transfer funds to another account for the currently logged-in user
    void transfer() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        string destAcc;
        while (true) {
            cout << "Destination Account Number (or press 0 to go back): ";
            cin >> destAcc;
            if (destAcc == "0") {
                cout << "Returning to menu." << endl;
                return;
            }
            BankAccount* destAccount = nullptr;
            for (auto& acc : accounts) {
                if (acc.getAccountNumber() == destAcc) {
                    destAccount = &acc;
                    break;
                }
            }
            if (!destAccount) {
                cout << "Destination account not found! Please try again (or press 0 to go back):" << endl;
                continue;
            }
            double amount;
            cout << "Amount to transfer (or press 0 to go back): ";
            cin >> amount;
            if (amount == 0) {
                cout << "Returning to menu." << endl;
                return;
            }
            currentAccount->transfer(*destAccount, amount);
            saveAccountsToFile();
            break;
        }
    }

    // Display account information for the current user
    void displayAccount() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        while (true) {
            currentAccount->display();
            cout << "Press 0 to go back: ";
            string input;
            cin >> input;
            if (input == "0") {
                cout << "Returning to menu." << endl;
                break;
            }
        }
    }

    // Edit the account holder's name for the current user
    void editAccountHolderName() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        cout << "Current Name & Surname: " << currentAccount->getNameSurname() << endl;
        string newName;
        while (true) {
            cout << "Enter new Name & Surname (or press 0 to go back): ";
            cin.ignore();
            getline(cin, newName);
            if (newName == "0") {
                cout << "Returning to menu." << endl;
                return;
            }
            if (newName.find(' ') == string::npos) {
                cout << "Please enter both name and surname (e.g. John Smith)." << endl;
                continue;
            }
            break;
        }
        currentAccount->setNameSurname(newName);
        saveAccountsToFile();
        cout << "Name & Surname updated successfully!" << endl;
    }

    // Edit the account email for the current user (with OTP verification)
    void editAccountEmail() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        cout << "Current Account Email: " << currentAccount->getEmail() << endl;
        string newEmail;
        while (true) {
            cout << "Enter new Account Email (or press 0 to go back): ";
            cin >> newEmail;
            if (newEmail == "0") {
                cout << "Returning to menu." << endl;
                return;
            }
            bool emailUsedByOther = false;
            for (const auto& acc : accounts) {
                if (acc.getEmail() == newEmail && acc.getAccountNumber() != currentAccount->getAccountNumber()) {
                    emailUsedByOther = true;
                    break;
                }
            }
            if (emailUsedByOther) {
                cout << "The email you entered is either already associated with another bank account or it doesn't exist. Please try again." << endl;
                continue;
            }
            int otpResult = verifyEmailOTP(newEmail, currentAccount->getNameSurname());
            if (otpResult == 1) {
                break;
            } else if (otpResult == -3) {
                cout << "The email you entered does not exist or is invalid. Please try again." << endl;
                continue;
            } else if (otpResult == -2) {
                cout << "OTP expired." << endl;
                cout << "Please check that your email address is correct and try again!" << endl;
            } else if (otpResult == -1) {
                cout << "Returning to menu." << endl;
                return;
            } else {
                cout << "Please check that your email address is correct and try again!" << endl;
            }
        }
        currentAccount->setEmail(newEmail);
        saveAccountsToFile();
        cout << "Account email updated successfully!" << endl;
    }

    // Edit the account password for the current user (with OTP verification)
    void editAccountPassword() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        string oldPwd, newPwd;
        cout << "Enter current password (or press 0 to go back): ";
        cin >> oldPwd;
        if (oldPwd == "0") {
            cout << "Returning to menu." << endl;
            return;
        }
        if (currentAccount->getPasswordHash() != hashPassword(oldPwd)) {
            cout << "Incorrect current password." << endl;
            return;
        }
        string email = currentAccount->getEmail();
        if (!verifyEmailOTP(email, currentAccount->getNameSurname())) {
            cout << "Password update aborted due to failed OTP verification." << endl;
            return;
        }
        do {
            cout << "Enter new password (12-16 chars, 1 uppercase, 1 lowercase, 1 digit, 1 special !@#$%^&*): ";
            cin >> newPwd;
            if (newPwd == "0") {
                cout << "Returning to menu." << endl;
                return;
            }
            if (!isStrongPassword(newPwd)) {
                cout << "Weak password! Please follow the password rules." << endl;
                continue;
            }
            break;
        } while (true);
        currentAccount->setPasswordHash(hashPassword(newPwd));
        saveAccountsToFile();
        cout << "Password updated successfully!" << endl;
    }

    // Delete the current user's account (with OTP verification)
    void deleteAccount() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        string confirm;
        cout << "Are you sure you want to delete your account? This action cannot be undone. (yes/no): ";
        cin >> confirm;
        if (confirm != "yes") {
            cout << "Account deletion cancelled." << endl;
            return;
        }
        string email = currentAccount->getEmail();
        cout << "To delete your account, you must verify with an OTP sent to your email." << endl;
        if (verifyEmailOTP(email, currentAccount->getNameSurname()) != 1) {
            cout << "Account deletion aborted due to failed OTP verification." << endl;
            return;
        }
        auto it = remove_if(accounts.begin(), accounts.end(),
            [&](const BankAccount& acc) { return acc.getAccountNumber() == currentAccount->getAccountNumber(); });
        accounts.erase(it, accounts.end());
        saveAccountsToFile();
        string txFile = currentAccount->getAccountNumber() + "_transactions.txt";
        remove(txFile.c_str());
        cout << "Account deleted successfully." << endl;
        currentAccount = nullptr;
    }

    // Check if an account exists by account number
    bool accountExists(const string& accountNumber) const {
        for (const auto& account : accounts) {
            if (account.getAccountNumber() == accountNumber) {
                return true;
            }
        }
        return false;
    }

    // Check if an email is already used by another account
    bool emailExists(const string& emailToCheck) const {
        for (const auto& account : accounts) {
            if (account.getEmail() == emailToCheck) {
                return true;
            }
        }
        return false;
    }

    // Check if a username is already used by another account
    bool usernameExists(const string& usernameToCheck) const {
        for (const auto& account : accounts) {
            if (account.getUsername() == usernameToCheck) {
                return true;
            }
        }
        return false;
    }

    // Check if an account is currently locked due to failed login attempts
    bool isAccountLocked(const string& accountNumber) {
        auto it = loginAttemptsMap.find(accountNumber);
        if (it != loginAttemptsMap.end()) {
            int attempts = it->second.first;
            auto lockTime = it->second.second;
            if (attempts >= MAX_ATTEMPTS) {
                auto now = chrono::system_clock::now();
                auto secondsPassed = chrono::duration_cast<chrono::seconds>(now - lockTime).count();
                if (secondsPassed < LOCK_DURATION_SECONDS) {
                    return true;
                } else {
                    loginAttemptsMap[accountNumber] = {0, chrono::system_clock::time_point()};
                    return false;
                }
            }
        }
        return false;
    }

    // Record a failed login attempt for an account
    void recordFailedAttempt(const string& accountNumber) {
        auto now = chrono::system_clock::now();
        auto& entry = loginAttemptsMap[accountNumber];
        entry.first += 1;
        if (entry.first >= MAX_ATTEMPTS) {
            entry.second = now;
        }
    }

    // Reset failed login attempts for an account
    void resetAttempts(const string& accountNumber) {
        loginAttemptsMap[accountNumber] = {0, chrono::system_clock::time_point()};
    }

    // Get the unlock time for a locked account
    time_t getUnlockTime(const string& accountNumber) const {
        auto it = loginAttemptsMap.find(accountNumber);
        if (it != loginAttemptsMap.end()) {
            int attempts = it->second.first;
            auto lockTime = it->second.second;
            if (attempts >= MAX_ATTEMPTS) {
                auto unlockTime = lockTime + chrono::seconds(LOCK_DURATION_SECONDS);
                return chrono::system_clock::to_time_t(unlockTime);
            }
        }
        return 0;
    }

    // Get the email of the currently logged-in account
    string getCurrentAccountEmail() const {
        if (currentAccount) return currentAccount->getEmail();
        return "";
    }

    // Verify an email address by sending and checking an OTP
    int verifyEmailOTP(const string& email, const string& nameSurname = "") {
        string otp = generateOTP();
        string command = "python send_otp_email.py \"" + email + "\" \"" + otp + "\" \"" + nameSurname + "\"";
        int result = system(command.c_str());

        if (result == 2 * 256) {
            return -3;
        }
        if (result != 0) {
            return -1;
        }

        cout << "Enter the OTP to verify your email - You have 1 minute and 3 attempts (or press 0 to go back): ";

        string enteredOtp;
        auto start = chrono::steady_clock::now();
        int otpAttempts = 0;
        const int MAX_OTP_ATTEMPTS = 3;

        while (otpAttempts < MAX_OTP_ATTEMPTS) {
            cin >> enteredOtp;
            if (enteredOtp == "0") {
                cout << "Returning to menu." << endl;
                return -1;
            }
            auto now = chrono::steady_clock::now();
            auto elapsed = chrono::duration_cast<chrono::seconds>(now - start).count();
            if (elapsed > 60) {
                return -2;
            }
            if (enteredOtp == otp) {
                cout << "Email verification successful." << endl;
                return 1;
            } else {
                otpAttempts++;
                if (otpAttempts < MAX_OTP_ATTEMPTS) {
                    cout << "Invalid OTP. Try again (or press 0 to go back). (" << otpAttempts << "/" << MAX_OTP_ATTEMPTS << " attempts)" << endl;
                }
            }
        }
        cout << "Too many failed OTP attempts." << endl;
        return -1;
    }

    // Apply interest to all accounts if today is an interest day
    void applyInterestToAllAccounts() {
        time_t now = time(nullptr);
        tm* now_tm = localtime(&now);

        bool isInterestDay =
            (now_tm->tm_mon + 1 == INTEREST_MONTH_1 && now_tm->tm_mday == INTEREST_DAY_1) ||
            (now_tm->tm_mon + 1 == INTEREST_MONTH_2 && now_tm->tm_mday == INTEREST_DAY_2);

        if (!isInterestDay) return;

        for (auto& account : accounts) {
            time_t lastApplied = account.getLastInterestApplied();
            tm* last_tm = localtime(&lastApplied);
            if (lastApplied == 0 ||
                last_tm->tm_year != now_tm->tm_year ||
                last_tm->tm_mon != now_tm->tm_mon ||
                last_tm->tm_mday != now_tm->tm_mday) {
                double oldBalance = account.getBalance();
                double interest = oldBalance * INTEREST_RATE;
                account.setBalance(oldBalance + interest);
                account.setLastInterestApplied(now);
                account.addTransaction(Transaction("INTEREST_APPLIED", interest));
            }
        }
        saveAccountsToFile();
    }

    // Display the transaction history for the current user
    void displayTransactionHistory() {
        if (!currentAccount) {
            cout << "No user logged in!" << endl;
            return;
        }
        const auto& txs = currentAccount->getTransactions();
        if (txs.empty()) {
            cout << "No transactions found." << endl;
            cout << "Press 0 to go back: ";
            string input;
            cin >> input;
            return;
        }

        for (const auto& t : txs) {
            cout << "[" << t.datetime << "] ";
            cout << t.type << " ";
            cout << fixed << setprecision(2) << t.amount << " $";
            if (!t.otherAccount.empty())
                cout << " (Other Account: " << t.otherAccount << ")";
            cout << endl;
        }
        cout << "Press 0 to go back: ";
        string input;
        cin >> input;
    }

    // Return a reference to all accounts (used for login and lookups)
    vector<BankAccount>& getAccounts() { return accounts; }

    // Set the currentAccount pointer for OTP login (no password check)
    void forceLogin(BankAccount* account) {
        currentAccount = account;
    }

private:
    // Save all accounts to the accounts file
    void saveAccountsToFile() {
        ofstream outFile(ACCOUNTS_FILE);
        if (!outFile) {
            cerr << "Error saving accounts!" << endl;
            return;
        }

        for (const auto& account : accounts) {
            outFile << account.getAccountNumber() << "|"
                    << account.getUsername() << "|"
                    << account.getPasswordHash() << "|"
                    << fixed << setprecision(2) << account.getBalance() << "|"
                    << account.getEmail() << "|"
                    << account.getNameSurname() << "|"
                    << account.getLastInterestApplied() << "\n";
        }
        outFile.close();
    }

    // Load all accounts from the accounts file
    void loadAccountsFromFile() {
        ifstream inFile(ACCOUNTS_FILE);
        if (!inFile) {
            return;
        }

        accounts.clear();
        string line;
        while (getline(inFile, line)) {
            stringstream ss(line);
            string number, username, pwdHash, balanceStr, email, nameSurname, lastInterestStr;

            getline(ss, number, '|');
            getline(ss, username, '|');
            getline(ss, pwdHash, '|');
            getline(ss, balanceStr, '|');
            getline(ss, email, '|');
            getline(ss, nameSurname, '|');
            getline(ss, lastInterestStr, '|');

            try {
                double balance = stod(balanceStr);
                time_t lastInterest = lastInterestStr.empty() ? 0 : stoll(lastInterestStr);
                accounts.emplace_back(number, username, pwdHash, balance, email, nameSurname, lastInterest);
                accounts.back().loadTransactionsFromFile();
            } catch (...) {
                cerr << "Error reading account data for account: " << number << endl;
            }
        }
        inFile.close();
    }
};

// Display the main menu for the banking system
void displayMainMenu() {
    cout << "\n\n=== THE BEST BANKING SYSTEM ===\n" << endl;
    cout << "1. Login" << endl;
    cout << "2. Create New Account" << endl;
    cout << "3. Contact Information" << endl;
    cout << "4. Privacy Policy" << endl;
    cout << "0. Exit" << endl;
    cout << "Choice: ";
}

// Display the menu for logged-in users
void displayUserMenu() {
    cout << "\n\n=== USER MENU ===\n" << endl;
    cout << "1. Deposit Amount" << endl;
    cout << "2. Withdraw Amount" << endl;
    cout << "3. Transfer Amount" << endl;
    cout << "4. Show Account Information" << endl;
    cout << "5. Account-Management" << endl;
    cout << "6. Transaction History" << endl;
    cout << "0. Logout" << endl;
    cout << "Choice: ";
}

// Display the account management menu for editing user details
void displayAccountManagementMenu() {
    cout << "\n\n=== ACCOUNT MANAGEMENT ===\n" << endl;
    cout << "1. Edit Account Holder Name" << endl;
    cout << "2. Edit Account Email" << endl;
    cout << "3. Edit Account Password" << endl;
    cout << "4. Delete Account" << endl;
    cout << "0. Back" << endl;
    cout << "Choice: ";
}

// Display contact information for the bank
void displayContactInfo() {
    cout << "\n\n=== CONTACT INFORMATION ===\n" << endl;
    cout << "Phone: +30 210 1234567" << endl;
    cout << "Email: support@thebestbankingsystem.gr" << endl;
    cout << "Address: Leoforos Kifisias 100, Marousi, 15125" << endl;
    cout << "Facebook: facebook.com/thebestbankingsystem" << endl;
    cout << "Service Hours: 24/7 support" << endl;
    cout << endl;
    cout << "Press 0 to go back: ";
    string input;
    cin >> input;
}

// Display the privacy policy for the banking system
void displayPrivacyPolicy() {
    cout << "\n\n=== PRIVACY POLICY ===\n" << endl;
    cout << " 1. What data we collect" << endl;
    cout << "When using the Banking System, we may collect the following data:" << endl;
    cout << "- Full name" << endl;
    cout << "- Email address" << endl;
    cout << "- Username" << endl;
    cout << "- Password (encrypted)" << endl;
    cout << "- Account balance" << endl;
    cout << "- Transaction history" << endl;
    cout << "- IP address and timestamps\n" << endl;
    cout << " 2. How we use your data" << endl;
    cout << "Your data is used exclusively for:" << endl;
    cout << "- Authenticating your identity during login" << endl;
    cout << "- Executing banking transactions" << endl;
    cout << "- Maintaining a secure transaction record" << endl;
    cout << "- Enhancing system security\n" << endl;
    cout << " 3. Who has access to your data" << endl;
    cout << "Your data remains strictly confidential and is not shared with third parties. Only authorized system administrators have access for management and support purposes.\n" << endl;
    cout << " 4. How we protect your data" << endl;
    cout << "- All sensitive data (such as passwords) are stored securely using encryption." << endl;
    cout << "- Access to the system requires user authentication." << endl;
    cout << "- The system logs suspicious access attempts and notifies the administrator.\n" << endl;
    cout << " 5. Cookies" << endl;
    cout << "The system may use technical cookies for session identification and to improve user experience. No cookies are used for advertising purposes.\n" << endl;
    cout << " 6. Your rights" << endl;
    cout << "As a user, you have the following rights:" << endl;
    cout << "- Request access to your data" << endl;
    cout << "- Request correction or deletion of your data" << endl;
    cout << "- Request a copy of your personal information" << endl;
    cout << "For any request, please contact the system administrator.\n" << endl;
    cout << " 7. Changes to the policy" << endl;
    cout << "The privacy policy may be updated periodically. The last modification was on [01/04/2025].\n" << endl;
    cout << " 8. Contact" << endl;
    cout << "For questions about the privacy policy, you can contact us at:" << endl;
    cout << " email@bankingsystem.gr" << endl;
    cout << " +30 210 1234567 (24/7 support)" << endl;
    cout << endl;
    cout << "Press 0 to go back: ";
    string input;
    cin >> input;
}

// Helper function for OTP login via email
BankAccount* loginWithOTP(vector<BankAccount>& accounts) {
    string email;
    cout << "Enter your account email: ";
    cin >> email;
    for (auto& acc : accounts) {
        if (acc.getEmail() == email) {
            string otp = generateOTP();
            sendOTPEmail(email, otp, acc.getNameSurname());
            cout << "Enter the OTP to verify your email (or press 0 to go back): ";
            string enteredOtp;
            auto start = chrono::steady_clock::now();
            int otpAttempts = 0;
            const int MAX_OTP_ATTEMPTS = 3;
            while (otpAttempts < MAX_OTP_ATTEMPTS) {
                cin >> enteredOtp;
                if (enteredOtp == "0") {
                    cout << "Returning to main menu." << endl;
                    return nullptr;
                }
                auto now = chrono::steady_clock::now();
                auto elapsed = chrono::duration_cast<chrono::seconds>(now - start).count();
                if (elapsed > 60) {
                    cout << "OTP expired. Please try again." << endl;
                    return nullptr;
                }
                if (enteredOtp == otp) {
                    cout << "OTP verification successful. Logging you in..." << endl;
                    return &acc;
                } else {
                    otpAttempts++;
                    if (otpAttempts < MAX_OTP_ATTEMPTS) {
                        cout << "Invalid OTP. Try again (or press 0 to go back). (" << otpAttempts << "/" << MAX_OTP_ATTEMPTS << " attempts)" << endl;
                    }
                }
            }
            cout << "Too many failed OTP attempts." << endl;
            return nullptr;
        }
    }
    cout << "No account found with this email." << endl;
    return nullptr;
}

// Unified login flow for username/password or email/OTP
BankAccount* unifiedLogin(vector<BankAccount>& accounts, bool& loggedIn) {
    int attempts = 0;
    const int MAX_ATTEMPTS = 3;
    while (attempts < MAX_ATTEMPTS) {
        cout << "Enter your Username:" << endl;
        cout << "(Press 1 if you forgot your Username, or 0 to go back) \n> ";
        string username;
        cin >> username;
        if (username == "0") return nullptr;
        BankAccount* userAccount = nullptr;
        if (username == "1") {
            userAccount = loginWithOTP(accounts);
            if (userAccount) {
                loggedIn = true;
                return userAccount;
            }
            attempts++;
            continue;
        }
        for (auto& acc : accounts) {
            if (acc.getUsername() == username) {
                userAccount = &acc;
                break;
            }
        }
        if (!userAccount) {
            attempts++;
            cout << "Username not found. Please try again (" << attempts << "/" << MAX_ATTEMPTS << " attempts)" << endl;
            continue;
        }

        // Password or OTP for password recovery
        int passAttempts = 0;
        while (passAttempts < MAX_ATTEMPTS) {
            cout << "Enter your Password:" << endl;
            cout << "(Press 1 if you forgot your Password, or 0 to go back) \n> ";
            string password;
            cin >> password;
            if (password == "0") return nullptr;
            if (password == "1") {
                userAccount = loginWithOTP(accounts);
                if (userAccount) {
                    loggedIn = true;
                    return userAccount;
                }
                passAttempts++;
                continue;
            }
            if (userAccount->getPasswordHash() == hashPassword(password)) {
                // 2FA OTP after password
                string otp = generateOTP();
                sendOTPEmail(userAccount->getEmail(), otp, userAccount->getNameSurname());
                cout << "\n\n=== LOGIN: OTP VERIFICATION ===\n" << endl;
                cout << "Enter the OTP to complete login - You have 1 minute and 3 attempts (or press 0 to go back): ";
                string enteredOtp;
                auto start = chrono::steady_clock::now();
                int otpAttempts = 0;
                while (otpAttempts < MAX_ATTEMPTS) {
                    cin >> enteredOtp;
                    if (enteredOtp == "0") return nullptr;
                    auto now = chrono::steady_clock::now();
                    auto elapsed = chrono::duration_cast<chrono::seconds>(now - start).count();
                    if (elapsed > 60) {
                        cout << "OTP expired. Please login again." << endl;
                        break;
                    }
                    if (enteredOtp == otp) {
                        loggedIn = true;
                        cout << "2FA successful. You are now logged in." << endl;
                        return userAccount;
                    } else {
                        otpAttempts++;
                        if (otpAttempts < MAX_ATTEMPTS) {
                            cout << "Invalid OTP. Try again (or press 0 to go back). (" << otpAttempts << "/" << MAX_ATTEMPTS << " attempts)" << endl;
                        }
                    }
                }
                cout << "Too many failed OTP attempts." << endl;
                return nullptr;
            } else {
                passAttempts++;
                cout << "Login failed! (" << passAttempts << "/" << MAX_ATTEMPTS << " attempts)" << endl;
            }
        }
        attempts++;
    }
    cout << "Too many failed attempts. Try again later." << endl;
    return nullptr;
}

// Main function: Entry point of the banking system application
int main() {
    BankSystem bank;
    bank.applyInterestToAllAccounts();
    int choice;

    do {
        displayMainMenu();
        cin >> choice;

        switch (choice) {
            case 1: { // Handles user login and authentication
                bool loggedIn = false;
                cout << "\n\n=== LOGIN ===\n" << endl;
                BankAccount* userAccount = unifiedLogin(bank.getAccounts(), loggedIn);
                if (!userAccount) {
                    cout << "Login aborted." << endl;
                    break;
                }
                bank.forceLogin(userAccount);

                // Check if the account is locked due to failed login attempts
                if (bank.isAccountLocked(userAccount->getAccountNumber())) {
                    time_t unlock_time_t = bank.getUnlockTime(userAccount->getAccountNumber());
                    cout << "This account is locked due to too many failed login attempts." << endl;
                    if (unlock_time_t != 0) {
                        cout << "You can try again at: " << put_time(localtime(&unlock_time_t), "%Y-%m-%d %H:%M:%S") << endl;
                    }
                    break;
                }

                int userChoice;
                do {
                    displayUserMenu();
                    cin >> userChoice;
                    switch (userChoice) {
                        case 1:
                            cout << "\n\n=== DEPOSIT AMOUNT ===\n" << endl;
                            bank.deposit();
                            break;
                        case 2:
                            cout << "\n\n=== WITHDRAW AMOUNT ===\n" << endl;
                            bank.withdraw();
                            break;
                        case 3:
                            cout << "\n\n=== TRANSFER AMOUNT ===\n" << endl;
                            bank.transfer();
                            break;
                        case 4:
                            cout << "\n\n=== ACCOUNT INFORMATION ===\n" << endl;
                            bank.displayAccount();
                            break;
                        case 5: {
                            int accMgmtChoice;
                            do {
                                displayAccountManagementMenu();
                                cin >> accMgmtChoice;
                                switch (accMgmtChoice) {
                                    case 1:
                                        cout << "\n\n--- EDIT ACCOUNT HOLDER NAME ---\n" << endl;
                                        bank.editAccountHolderName();
                                        break;
                                    case 2:
                                        cout << "\n\n--- EDIT ACCOUNT EMAIL ---\n" << endl;
                                        bank.editAccountEmail();
                                        break;
                                    case 3:
                                        cout << "\n\n--- EDIT ACCOUNT PASSWORD ---\n" << endl;
                                        bank.editAccountPassword();
                                        break;
                                    case 4:
                                        cout << "\n\n--- DELETE ACCOUNT ---\n" << endl;
                                        bank.deleteAccount();
                                        accMgmtChoice = 0;
                                        break;
                                    case 0:
                                        cout << "Returning to main menu." << endl;
                                        break;
                                    default:
                                        cout << "Invalid choice!" << endl;
                                }
                                // If account was deleted, exit management menu
                                if (bank.getCurrentAccountEmail() == "") break;
                            } while (accMgmtChoice != 0);
                            break;
                        }
                        case 6:
                            cout << "\n\n=== TRANSACTION HISTORY ===\n" << endl;
                            bank.displayTransactionHistory();
                            break;
                        case 0:
                            bank.logout();
                            break;
                        default:
                            cout << "Invalid choice!" << endl;
                    }
                    // If account was deleted or logged out, exit user menu
                    if (bank.getCurrentAccountEmail() == "") break;
                } while (userChoice != 0);
                break;
            }
            case 2:
                cout << "\n\n=== CREATE NEW ACCOUNT ===\n" << endl;
                bank.createAccount();
                break;
            case 3:
                displayContactInfo();
                break;
            case 4:
                displayPrivacyPolicy();
                break;
            case 0:
                cout << "Exiting..." << endl;
                break;
            default:
                cout << "Invalid choice!" << endl;
        }
    } while (choice != 0);

    return 0;
}
