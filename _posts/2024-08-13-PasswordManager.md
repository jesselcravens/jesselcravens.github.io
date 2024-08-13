---
layout: post
title:  My own local Password Manager
date:   2024-08-13 14:10:32 -0500
categories: Python
---
# My Own Local Password Manager

Foe personal use and for work, I find that I really do need a password manager. I also have a distrust for using random web services to store sensitive information. My workplace for a period of time did not have an official password manager, so I decided to make my own to not violate any upcoming policies and for the challenge.

In my own naming convention for my personal tools, this is named PassiGUI. 


Internal Design Elements: 

- Uses SqlLite to store credentials
- All credentials and master password are encrypted 
- Application is compiled into one exe file


Screenshots: 

![](/assets/PassiGUI1.PNG)
![](/assets/PassiGUI2.PNG)
![](/assets/PassiGUI3.PNG)
![](/assets/PassiGUI4.PNG)
![](/assets/PassiGUI5.PNG)
![](/assets/PassiGUI6.PNG)
![](/assets/PassiGUI7.PNG)

Source code : 

Main Application: 

```python
import tkinter as tk
from tkinter import font as tkfont
from os import path
from tkinter import *
from tkinter import ttk
from tkinter import messagebox
from cryptography.fernet import Fernet
import sqlite3
from sqlite3 import Error
from string import ascii_letters, digits
import random

class MainFrame(tk.Tk):
    """The MainFrame Class controls the container for the other two pages in the application"""

    code_version = 'Version 1: Last Modified 1/03/24'
    def __init__(self,   *args, **kwargs):
        tk.Tk.__init__(self, *args, **kwargs)

        #define the container frame
        container = tk.Frame()
        container.grid(row=0, column=0, sticky='nsew' )

        #Basic properties of the main window
        self.titlefont = tkfont.Font(family='Verdana', size=10, weight='normal', slant='roman')
        self.wm_title('PassiGUI')
        self.wm_resizable(True, True)
        path_to_icon = path.abspath(path.join(path.dirname(__file__), 'green_lock.ico'))
        self.wm_iconbitmap(path_to_icon)

        # Please generate a new key from cred_work.py for each new install
        self.key = b'<change me>'

        self.id = tk.StringVar()
        self.id.set("Main")


        #Define a dictionary to hold details about the frames in the application
        self.listing = {}


        #insert details in the frames dictionary
        for p in (Security, Create_window, FirstRun, PassiGUI):
            page_name = p.__name__
            frame = p(parent=container, controller=self)
            frame.grid(row=0, column=0, sticky='nsew')
            self.listing[page_name] = frame

        #start with the first Frame BlobiGUI
        self.up_frame('FirstRun')
        self.first_run()

    def first_run(self):
        """When the main window is loaded, this function will be called to create the SQLlite DB and tables"""
        self.sqllite_create_table()
        self.sqllite_create_master_password()

    def sqllite_query(self, query_text):
        """Generic Sqllite query function to be called for other functions"""
        try:
            conn = sqlite3.connect('passigui.db')
            query = query_text
            cursor = conn.cursor()
            result = conn.execute(query)
            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
        finally:
            if conn:
                conn.commit()
                conn.close()
    def sqllite_create_table(self):
        """Creates the SQLlite credentials table to store"""
        query_text = """CREATE TABLE IF NOT EXISTS credentials(cred_name TEXT PRIMARY KEY NOT NULL, value TEXT);"""
        self.sqllite_query(query_text=query_text)
    def sqllite_create_master_password(self):

        query_text =  """CREATE TABLE IF NOT EXISTS password(master TEXT NOT NULL);"""
        self.sqllite_query(query_text=query_text)

    def decrypt_value_string(self, value):
        mainkey = self.key

        cipher_suite = Fernet(mainkey)
        flat_value = value[0]
        string_value = flat_value[1:]
        trim_value = string_value.replace("'", "")
        bytes_value = bytes(trim_value, "utf-8")
    #
        unciphered_text = cipher_suite.decrypt(bytes_value)
        plain_text_creds = bytes(unciphered_text).decode("utf-8")
        return plain_text_creds

    def encrypt_value_string(self, value):
        mainkey = self.key
        cipher_suite = Fernet(mainkey)

        binary_combined_creds = value.encode()
        ciphered_creds = cipher_suite.encrypt(binary_combined_creds)

        return ciphered_creds


    def up_frame(self, page_name):
        """This function take a page name and raises the frame to the top. A frame on the bottom is not visible.
        This is how page switching in the application is accomplished"""
        page = self.listing[page_name]
        page.tkraise()

class PassiGUI(tk.Frame):
    """This is class for the main application page. This handles viewing and editing of stored creds """

    def __init__(self, parent, controller):
        # Main Window Options

        tk.Frame.__init__(self, parent)

        #This links the controller to the class for frame control.
        self.controller = controller
        self.id = controller.id

        #UI design section for the class

        #--------------------------------------------------------------------------------------------------------------#
        # Top frame with the list box
        self.frame_top = ttk.Frame(self)
        self.frame_top.pack()

        self.CredListBox = Listbox(self.frame_top, width=50, height=20)

        cred_list = self.sqllite_read_cred_names()
        for i in cred_list:
            self.CredListBox.insert(END, i)
        self.CredListBox.grid(row=0, column=0)

        #--------------------------------------------------------------------------------------------------------------#
        # Frame with command buttons
        self.frame_load_command = ttk.Frame(self)
        self.frame_load_command.pack()

        self.load_button = ttk.Button(self.frame_load_command, text = "Load Credential Details",
                                      command=lambda:self.load_cred_deets())
        self.load_button.grid(row=0, column=0, padx=10, pady=10, ipadx=5, ipady=5)

        self.frame_commands = ttk.Frame(self)
        self.frame_commands.pack()
        self.new_button = ttk.Button(self.frame_commands, text = "New", command=lambda:self.create_new_cred())
        self.new_button.grid(row=1, column=1)
        self.refresh_list = ttk.Button(self.frame_commands, text = "Refresh List", command=lambda:self.refresh_names())
        self.refresh_list.grid(row=1, column=0)
        self.delete_button = ttk.Button(self.frame_commands, text = "Delete", command=lambda:self.delete_entry())
        self.delete_button.grid(row=1, column=2)


        #--------------------------------------------------------------------------------------------------------------#
        # Bottom frame with text box and change button
        self.frame_textwork = ttk.Frame(self)
        self.frame_textwork.pack()

        ttk.Label(self.frame_textwork, text = 'Pass Details').grid(row = 1, column = 0, sticky = 'nw', padx = 5)
        self.execute_text = Text(self.frame_textwork, width = 83, height = 15, wrap='word')
        xscroll1 = ttk.Scrollbar(self.frame_textwork, orient = HORIZONTAL, command = self.execute_text.xview)
        yscroll1 = ttk.Scrollbar(self.frame_textwork, orient = VERTICAL, command = self.execute_text.yview)
        self.execute_text.config(xscrollcommand = xscroll1.set, yscrollcommand = yscroll1.set)
        self.execute_text.grid(row = 2, column = 0, sticky = 'nw' , padx = 5)
        xscroll1.grid(row = 3, column = 0, sticky = 'ew')
        yscroll1.grid(row = 2, column = 1, sticky = 'ns')



        #This binds the right click mouse button to the RightClicker class with function for copy/paste/cut functions
        self.execute_text.bind("<Button-3>", RightClicker)

        #--------------------------------------------------------------------------------------------------------------#
        # Bottom frame with random and  change button
        self.frame_bottom_buttons = ttk.Frame(self)
        self.frame_bottom_buttons.pack()

        self.clear_button = ttk.Button(self.frame_bottom_buttons, text = 'Save Changes',
                                       command =lambda: self.edit_entry())
        self.clear_button.grid(row=1, column=1, sticky='nsew', padx=10, pady=10)
        self.random_button = ttk.Button(self.frame_bottom_buttons, text = 'Random Character',
                                       command =lambda: self.insert_random())
        self.random_button.grid(row=1, column=0, sticky='nsew', padx=10, pady=10)

    def refresh_names(self):
        """Delete the selection and cred text displayed to properly refresh """

        self.execute_text.delete(1.0, 'end')
        self.CredListBox.delete(0, 'end')

        new_list = self.sqllite_read_cred_names()

        for i in new_list:
            self.CredListBox.insert(END, i)

    def delete_entry(self):
        """This function will selected credential name and value with a confirmation message """
        credName = ''
        current_selection = self.CredListBox.curselection()
        if current_selection:
            for i in current_selection:
                credList = self.CredListBox.get(i)
                for x in credList:
                    confirm_messages = "Are you sure you want to delete credential " + str(x)
                    confirm_delete = messagebox.askyesno(title="Confirm Delete", message=confirm_messages)
                    print(confirm_delete)
                    if confirm_delete:
                        try:
                            conn = sqlite3.connect('passigui.db')
                            query = "Delete from credentials where cred_name = '" + str(x) + "';"

                            cursor = conn.cursor()
                            result = conn.execute(query)
                            cursor.close()
                        except sqlite3.Error as error:
                            print("Error", error)
                            messagebox.showerror(title="Error encountered", message=error)
                        finally:
                            if conn:
                                conn.commit()
                                conn.close()

    def edit_entry(self):
        """This function will update the contents of a stored credential based on changed data in the text window"""
        credName = ''
        current_details = self.execute_text.get(1.0, END)
        encoded_value = self.controller.encrypt_value_string(value=current_details)

        current_selection = self.CredListBox.curselection()
        if current_selection:
            for i in current_selection:
                credList = self.CredListBox.get(i)
                for x in credList:
                    confirm_messages = "Are you sure you want to overwrite credential details? " + str(x)
                    confirm_edit = messagebox.askyesno(title="Confirm Edit", message=confirm_messages)

                    if confirm_edit:
                        try:
                            conn = sqlite3.connect('passigui.db')
                            query = """UPDATE credentials SET value = """ + '"' + str(encoded_value) + '"' +  \
                                    " WHERE cred_name = '" + str(x) + "';"
                            cursor = conn.cursor()
                            result = conn.execute(query)
                            cursor.close()
                        except sqlite3.Error as error:
                            print("Error", error)
                            messagebox.showerror(title="Error encountered", message=error)
                        finally:
                            if conn:
                                conn.commit()
                                conn.close()


    def load_cred_deets(self):
        """This function will consider the highlighted cred and insert the decoded results into the textbox"""
        credName = ''
        self.execute_text.delete(1.0, 'end')
        current_selection = self.CredListBox.curselection()
        if current_selection:
            for i in current_selection:
                credList = self.CredListBox.get(i)
                for x in credList:
                    credName = x
                    details = self.sqllite_read_cred_deets(credName)
                    self.execute_text.insert(END, (str(details) + "\n"))
                    self.execute_text.see("end")

    def insert_random(self):
        """This will insert a random character/number into the current cursor position in the text box"""
        rando = random.choice(ascii_letters + digits)
        self.execute_text.insert("insert", rando)
    def create_new_cred(self):
        """Change the active window the cred creation window"""
        self.controller.up_frame(page_name="Create_window")

    def sqllite_read_cred_names(self):
        """This function will read the name of credentials from the SQLLite DB"""
        list_of_creds = []
        try:
            conn = sqlite3.connect('passigui.db')
            query = """Select cred_name from credentials;"""
            cursor = conn.cursor()
            result = conn.execute(query)
            for row in result:
                list_of_creds.append(row)

            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
        finally:
            if conn:
                conn.close()

        return list_of_creds

    def sqllite_read_cred_deets(self, cred_name):
        """This function will read and decode the credential values from the SQLlite DB"""
        cred_deets = ""
        cred_text = ''
        try:
            conn = sqlite3.connect('passigui.db')
            query = """Select value from credentials where cred_name = '""" + cred_name + "';"

            cursor = conn.cursor()
            result = conn.execute(query)
            for row in result:
                cred_deets = row
                cred_text = self.controller.decrypt_value_string(value=cred_deets)

            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
        finally:
            if conn:
                conn.close()

        return cred_text

class RightClicker():
    """This class makes it possible to right click in a text box and Cut/Copy/Paste"""
    def __init__(self, e):
        commands = ["Cut","Copy","Paste"]
        menu = tk.Menu(None, tearoff=0, takefocus=0)

        for txt in commands:
            menu.add_command(label=txt, command=lambda e=e,txt=txt:self.click_command(e,txt))

        menu.tk_popup(e.x_root + 40, e.y_root + 10, entry="0")

    def click_command(self, e, cmd):
        e.widget.event_generate(f'<<{cmd}>>')
class Security(tk.Frame):
    """This class is the security window a user would encounter to log into the application"""

    def __init__(self, parent, controller):
        tk.Frame.__init__(self, parent)
        self.controller = controller
        self.id = controller.id

        #--------------------------------------------------------------------------------------------------------------#
        # UI design section

        #--------------------------------------------------------------------------------------------------------------#
        # Top frame with words
        self.frame_top = ttk.Frame(self)
        self.frame_top.pack()


        ttk.Label(self.frame_top, text="Please enter your password to continue") \
            .grid(row=0, column=0, pady=15)

        #--------------------------------------------------------------------------------------------------------------#
        # Bottom frame with Password entry

        self.frame_entry = ttk.Frame(self)
        self.frame_entry.pack()

        ttk.Label(self.frame_entry, text="Password: ").grid(row=1, column=0, pady=15)


        self.passbox_value = StringVar()
        self.passbox = ttk.Entry(self.frame_entry, textvariable=self.passbox_value, show="*")
        self.passbox.grid(row=1, column=1, pady=15)
        self.check_button = ttk.Button(self.frame_entry, text="Check Password", command=lambda:self.check_password())
        self.check_button.grid(row=1, column=2)

    def sqllite_read_master_password(self):
        """This function reads and decodes the stored master password"""
        cred_deets = ""
        cred_text = ''
        try:
            conn = sqlite3.connect('passigui.db')
            query = """Select master from password Limit 1;"""

            cursor = conn.cursor()
            result = conn.execute(query)
            for row in result:
                cred_deets = row
                cred_text = self.controller.decrypt_value_string(value=cred_deets)

            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
        finally:
            if conn:
                conn.close()

        return cred_text

    def check_password(self):
        """This function compares the password in the DB to the entered password. If successful, the visible window
        will change to the main window"""
        entered_pass = self.passbox_value.get()
        real_pass = self.sqllite_read_master_password()
        if entered_pass == real_pass:
            self.controller.up_frame(page_name='PassiGUI')
        else:
            messagebox.showerror(message="Please try again. Or give up. That is always an option")

class Create_window(tk.Frame):
    """This class defines the credential creation page of the application."""

    def __init__(self, parent, controller):
        tk.Frame.__init__(self, parent)
        self.controller = controller
        self.id = controller.id

        #--------------------------------------------------------------------------------------------------------------#
        # UI design section

        #--------------------------------------------------------------------------------------------------------------#
        # Top Frame


        self.frame_top = ttk.Frame(self)
        self.frame_top.pack()

        #--------------------------------------------------------------------------------------------------------------#
        # UI Credential Name entry frame


        self.frame_name_entry = ttk.Frame(self)
        self.frame_name_entry.pack()

        self.new_cred_name = StringVar()
        self.new_cred_deets = StringVar()

        ttk.Label(self.frame_name_entry, text="Credential Name").grid(row=0, column=0)
        self.new_cred_name_entry = ttk.Entry(self.frame_name_entry, textvariable=self.new_cred_name)
        self.new_cred_name_entry.grid(row=0, column=1)

        #--------------------------------------------------------------------------------------------------------------#
        # Text work section


        self.frame_textwork = ttk.Frame(self)
        self.frame_textwork.pack()

        ttk.Label(self.frame_textwork, text = 'Pass Details').grid(row = 1, column = 0, sticky = 'nw', padx = 5)
        self.execute_text = Text(self.frame_textwork, width = 83, height = 15, wrap='word')
        xscroll1 = ttk.Scrollbar(self.frame_textwork, orient = HORIZONTAL, command = self.execute_text.xview)
        yscroll1 = ttk.Scrollbar(self.frame_textwork, orient = VERTICAL, command = self.execute_text.yview)
        self.execute_text.config(xscrollcommand = xscroll1.set, yscrollcommand = yscroll1.set)
        self.execute_text.grid(row = 2, column = 0, sticky = 'nw' , padx = 5)
        xscroll1.grid(row = 3, column = 0, sticky = 'ew')
        yscroll1.grid(row = 2, column = 1, sticky = 'ns')

        #--------------------------------------------------------------------------------------------------------------#
        # Frame with Bottom Commands buttons


        self.frame_bottom_buttons = ttk.Frame(self)
        self.frame_bottom_buttons.pack()

        self.execute_text.bind("<Button-3>", RightClicker)

        self.save_button = ttk.Button(self.frame_bottom_buttons, text = 'Save Changes',
                                       command =lambda: self.create_cred())
        self.save_button.grid(row=1, column=1, sticky='nsew', padx=10, pady=10)

        self.random_button = ttk.Button(self.frame_bottom_buttons, text = 'Random Character',
                                      command =lambda: self.insert_random())
        self.random_button.grid(row=1, column=2, sticky='nsew', padx=10, pady=10)

        self.GoBack_button = ttk.Button(self.frame_bottom_buttons, text = 'Go Back to Main Window',
                                       command =lambda: self.back_to_main())
        self.GoBack_button.grid(row=1, column=3, sticky='nsew', padx=10, pady=10)

    def create_cred(self):
        """This function will insert a credential and its associated value in the SQL lite DB"""
        value = self.execute_text.get(1.0, 'end')
        name = self.new_cred_name.get()
        encoded_value = self.controller.encrypt_value_string(value=value)

        try:
            conn = sqlite3.connect('passigui.db')
            query = """Insert into credentials(cred_name, value) Values(""" + '"' +  str(name) \
                    + '"' +  ", " + '"' + str(encoded_value) + '"' + ");"

            cursor = conn.cursor()
            result = conn.execute(query)
            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
            messagebox.showerror(title="Error encountered", message=error)
        finally:
            if conn:
                conn.commit()
                conn.close()


    def insert_random(self):
        """This function will insert a random character in the textbox"""
        rando = random.choice(ascii_letters + digits)
        self.execute_text.insert("insert", rando)

    def back_to_main(self):
        """Delete all the text fields and go back to the main window """
        self.execute_text.delete(1.0, 'end')
        self.new_cred_name_entry.delete(0, 'end')
        self.controller.up_frame(page_name='PassiGUI')

class FirstRun(tk.Frame):
    """This page of the application is the first stop used to detect the first run and set the master password"""

    def __init__(self, parent, controller):
        tk.Frame.__init__(self, parent)
        self.controller = controller
        self.id = controller.id

        #--------------------------------------------------------------------------------------------------------------#
        #UI design section

        #--------------------------------------------------------------------------------------------------------------#
        #UI Define top frames
        self.frame_top = ttk.Frame(self)
        self.frame_top.pack()

        self.frame_entry = ttk.Frame(self)
        self.frame_entry.pack()

        #--------------------------------------------------------------------------------------------------------------#
        # An IF statement to determine which fields to show if the master password is in the DB

        if self.sqllite_read_master_password() != []:
            ttk.Label(self.frame_top, text="Welcome Back!") \
                .grid(row=0, column=0, pady=15)
            self.redirect_to_security()
            self.login_button = ttk.Button(self.frame_entry, text="Click to Login",
                                           command=lambda:self.redirect_to_security())
            self.login_button.grid(row=2, column=1)


        else:
            ttk.Label(self.frame_top, text="Master Password NOT Detected. Please enter a new password ") \
                .grid(row=0, column=0, pady=15)
            ttk.Label(self.frame_entry, text="Password: ").grid(row=1, column=0, pady=15)
            ttk.Label(self.frame_entry, text="Confirm Password: ").grid(row=2, column=0, pady=15)


            self.passbox_1 = StringVar()
            self.passbox_1 = ttk.Entry(self.frame_entry, textvariable=self.passbox_1, show="*")
            self.passbox_1.grid(row=1, column=1, pady=15)

            self.passbox_2 = StringVar()
            self.passbox_2 = ttk.Entry(self.frame_entry, textvariable=self.passbox_1, show="*")
            self.passbox_2.grid(row=2, column=1, pady=15)

            self.makepw_button = ttk.Button(self.frame_entry, text="Set Password", command=lambda:self.set_password())
            self.makepw_button.grid(row=3, column=0, ipadx=10, ipady=10)


    def set_password(self):
        """This function is used to check if the two provided passwords match and set the master password"""
        password_1 = self.passbox_1.get()
        password_2 = self.passbox_2.get()


        if password_1 == password_2:
            success_message = "Password is set, please record this for future use: " + str(password_2)
            self.sqllite_create_master_password()
            #If the password is set, all credentials in the DB are deleted for safety
            self.sqllite_delete_all_creds()
            messagebox.showinfo(title="Successful Password Created", message=success_message)
            self.redirect_to_security()

        elif password_1 != password_2:
            messagebox.showwarning(title="Passwords do not match", message="Passwords do not match, please try again")

    def sqllite_create_master_password(self):
        """This function will set the master password in the DB"""
        master_pw = self.passbox_1.get()
        #The password is encrypted as stored in the DB
        encoded_value = self.controller.encrypt_value_string(value=master_pw)

        try:
            conn = sqlite3.connect('passigui.db')
            query = """Insert into password(master) Values(""" + '"' + str(encoded_value) + '"' + ");"

            cursor = conn.cursor()
            result = conn.execute(query)
            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
            messagebox.showerror(title="Error encountered", message=error)
        finally:
            if conn:
                conn.commit()
                conn.close()
    def sqllite_delete_all_creds(self):
        """This is a safety measure so that a user does not delete the master and set a new one
        to access all previous creds"""
        master_pw = self.passbox_1.get()
        encoded_value = self.controller.encrypt_value_string(value=master_pw)

        try:
            conn = sqlite3.connect('passigui.db')
            query = """delete from credentials; """

            cursor = conn.cursor()
            result = conn.execute(query)
            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
            messagebox.showerror(title="Error encountered", message=error)
        finally:
            if conn:
                conn.commit()
                conn.close()
    def sqllite_read_master_password(self):
        """This function reads the encoded master password and return it"""
        master_pw = []
        try:
            conn = sqlite3.connect('passigui.db')
            query = """Select master from password LIMIT 1;"""
            cursor = conn.cursor()
            result = conn.execute(query)
            for row in result:
                master_pw.append(row)

            cursor.close()
        except sqlite3.Error as error:
            print("Error", error)
        finally:
            if conn:
                conn.close()

        return master_pw

    def redirect_to_security(self):
        """This function changes the active window to the security login page"""
        self.controller.up_frame(page_name='Security')


# Tkinter App stuff
if __name__ == '__main__':
    app = MainFrame()
    app.mainloop()




# build instructions for py-installer: pyinstaller --windowed  --add-data "C:\Users\jcravens\IdeaProjects\PassiGUI\.idea\green_lock.ico;." --name PASSiGUI --onefile Main.py
```

Generating a Encryption String: 

```python
from cryptography.fernet import Fernet
print(Fernet.generate_key())
```

