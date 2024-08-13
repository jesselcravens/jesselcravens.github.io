---
layout: post
title:  Multiple Server Pull and Push Files Application
date:   2024-08-13 17:56:32 -0500
categories: Python
---
# Multiple Server Pull and Push Files Application



This is a tool I created to help manage manual backups and upgrades of multiple Linux servers at once. I call the tool PulliGUI based on its first function which was to only pull backup files. 



Capabilities:

- Can currently support up to 60 servers at once
- Flexible filenames and paths for files to pull from the servers
- Ability to push files to servers
- Ability to open the IP addresses selected in a web browser.  This is nice for servers with Web based user interfaces.  
- Flexible user name and passwords so that nothing is hard coded for security. 



Screenshot:

![](/assets/PulliGUI.PNG)

Code for the interested: 

```python
import os
import tkinter as tk
from tkinter import font as tkfont
import datetime
from os import path
from tkinter import *
from tkinter import ttk
from tkinter import filedialog
from tkinter import messagebox
import csv
import paramiko
from scp import SCPClient
import re
import os
import webbrowser


class MainFrame(tk.Tk):
    """The MainFrame Class controls the container for pages in the application"""

    code_version = 'Version 2: Last Modified 8/8/24'
    def __init__(self,   *args, **kwargs):
        tk.Tk.__init__(self, *args, **kwargs)

        #define the container frame
        container = tk.Frame()
        container.grid(row=0, column=0, sticky='nsew' )

        #Basic properties of the main window
        self.titlefont = tkfont.Font(family='Verdana', size=10, weight='normal', slant='roman')
        self.wm_title('PulliGUI')
        self.wm_resizable(True, True)
        path_to_icon = path.abspath(path.join(path.dirname(__file__), 'pulley.ico'))
        self.wm_iconbitmap(path_to_icon)

        self.id = tk.StringVar()
        self.id.set("Main")


        #Define a dictionary to hold details about the frames in the application
        self.listing = {}

        #insert details in the frames dictionary
        for p in (MainUI, Transfer):
            page_name = p.__name__
            frame = p(parent=container, controller=self)
            frame.grid(row=0, column=0, sticky='nsew')
            self.listing[page_name] = frame

        #start with the first Frame Transfer since this is currently only a one page application.
        self.up_frame('Transfer')



    def up_frame(self, page_name):
        """This function take a page name and raises the frame to the top. A frame on the bottom is not visible.
        This is how page switching in the application is accomplished"""
        page = self.listing[page_name]
        page.tkraise()

class MainUI(tk.Frame):
    """This is class for the main application page."""

    def __init__(self, parent, controller):
        # Main Window Options

        tk.Frame.__init__(self, parent)

        #This links the controller to the class for frame control.
        self.controller = controller
        self.id = controller.id


class Transfer(tk.Frame):
    """This class defines the main useful page of the application to handle a faster way to transfer files from
    many servers"""

    def __init__(self, parent, controller):
        tk.Frame.__init__(self, parent)
        self.controller = controller
        self.id = controller.id


        #UI design for Transfer window begins


        # First frame in the window with the title label, and the go back button
        self.frame_top = ttk.Frame(self)
        self.frame_top.columnconfigure(index=4, weight=1)
        self.frame_top.pack()


        self.frame_directory_selection = ttk.Frame(self)
        self.frame_directory_selection.pack()

        self.file_to_pull = StringVar()
        self.pull_location_entry = Entry(self.frame_directory_selection, textvariable=self.file_to_pull)
        self.pull_location_entry.grid(row = 1, column = 0, sticky = 'nw', padx = 5, pady = 5)
        ttk.Label(self.frame_directory_selection, text="Source location for filename to pull; example /etc/hosts").\
            grid(row = 1, column = 1, sticky = 'nw', padx = 5, pady = 5)

        self.transfer_target = StringVar()
        self.dir_entry = Entry(self.frame_directory_selection, textvariable=self.transfer_target )
        self.dir_entry.grid(row = 2, column = 0, sticky = 'nw', padx = 5, pady = 5)
        insert_select_dir_button = ttk.Button(self.frame_directory_selection, text = 'Select local Backup Directory for pulled files',
                                              command =lambda: self.select_directory())
        insert_select_dir_button.grid(row = 2, column = 1, sticky = 'w', padx = 5)



        self.push_file_target = StringVar()
        self.push_file_entry = Entry(self.frame_directory_selection, textvariable=self.push_file_target)
        self.push_file_entry.grid(row = 3, column = 0, sticky = 'nw', padx = 5, pady = 5)
        insert_select_push_file_button = ttk.Button(self.frame_directory_selection, text = 'Select File to Push',
                                                    command =lambda: self.select_push_file())
        insert_select_push_file_button.grid(row = 3, column = 1, sticky = 'w', padx = 5)

        self.push_location = StringVar()
        self.push_location_entry = Entry(self.frame_directory_selection, textvariable=self.push_location)
        self.push_location_entry.grid(row = 4, column = 0, sticky = 'nw', padx = 5, pady = 5)
        ttk.Label(self.frame_directory_selection, text = "Destination location for file to push; example /tmp/"). \
            grid(row = 4, column = 1, sticky = 'nw', padx = 5, pady = 5)

        self.server_username = StringVar()
        self.server_username_entry = Entry(self.frame_directory_selection, textvariable=self.server_username)
        self.server_username_entry.grid(row = 5, column = 0, sticky = 'nw', padx = 5, pady = 5)
        ttk.Label(self.frame_directory_selection, text = "Server Username").grid(row = 5, column = 1, sticky = 'nw', padx = 5, pady = 5)

        self.server_password = StringVar()
        self.server_password_entry = Entry(self.frame_directory_selection, show="*", textvariable=self.server_password)
        self.server_password_entry.grid(row = 6, column = 0, sticky = 'nw', padx = 5, pady = 5)
        ttk.Label(self.frame_directory_selection, text="Server Password").grid(row = 6, column = 1, sticky = 'nw', padx = 5, pady = 5)

        self.frame_csv_target = ttk.Frame(self)
        self.frame_csv_target.pack()

        ttk.Label(self.frame_csv_target, text = '').grid(row = 0, column = 0, sticky = 'nsew')
        ttk.Label(self.frame_csv_target, text = 'Select Servers to Transfer Files'
                                                ' (Options generated by server_list.csv'
                                                ' in program directory)').grid(row = 1, column = 0, sticky = 'nsew')



        # Frame with select and deselect all buttons
        self.frame_select_buttons = ttk.Frame(self)
        self.frame_select_buttons.pack()

        self.selectall_button = ttk.Button(self.frame_select_buttons, text = 'Select All',
                                           command =lambda: self.select_all_button())
        self.selectall_button.grid(row = 0, column = 0, sticky = 'w', padx = 5)
        self.deselectall_button = ttk.Button(self.frame_select_buttons, text = 'Deselect All',
                                             command =lambda: self.deselect_all_button())
        self.deselectall_button.grid(row = 0, column = 1, sticky = 'w', padx = 5)
        ttk.Label(self.frame_select_buttons, text = '').grid(row = 1, column = 0, sticky = 'nsew')



        #Frame with the Option boxes. There are a maximum of 40

        self.frame_selection = ttk.Frame(self)
        self.frame_selection.pack()

        self.state_aa_1 = IntVar()
        self.name_aa_1 = StringVar()
        self.ip_aa_1 = StringVar()
        self.select_aa_1 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_1, onvalue=1 , offvalue=0)
        self.select_aa_1.grid(row = 2, column = 0,sticky = 'ne' )
        self.name_box_aa_1 = Entry(self.frame_selection, textvariable=self.name_aa_1)
        self.name_box_aa_1.grid(row = 2, column = 1, sticky = 'nw')
        self.ip_box_aa_1 = Entry(self.frame_selection, textvariable=self.ip_aa_1)
        self.ip_box_aa_1.grid(row = 2, column = 2, sticky = 'nw')

        self.state_aa_2 = IntVar()
        self.name_aa_2 = StringVar()
        self.ip_aa_2 = StringVar()
        self.select_aa_2 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_2, onvalue=1 , offvalue=0)
        self.select_aa_2.grid(row = 2, column = 4,sticky = 'ne' )
        self.name_box_aa_2 = Entry(self.frame_selection, textvariable=self.name_aa_2)
        self.name_box_aa_2.grid(row = 2, column = 5, sticky = 'nw')
        self.ip_box_aa_2 = Entry(self.frame_selection, textvariable=self.ip_aa_2)
        self.ip_box_aa_2.grid(row = 2, column = 6, sticky = 'nw')

        self.state_aa_3 = IntVar()
        self.name_aa_3 = StringVar()
        self.ip_aa_3 = StringVar()
        self.select_aa_3 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_3, onvalue=1 , offvalue=0)
        self.select_aa_3.grid(row = 3, column = 0,sticky = 'ne' )
        self.name_box_aa_3 = Entry(self.frame_selection, textvariable=self.name_aa_3)
        self.name_box_aa_3.grid(row = 3, column = 1, sticky = 'nw')
        self.ip_box_aa_3 = Entry(self.frame_selection, textvariable=self.ip_aa_3)
        self.ip_box_aa_3.grid(row = 3, column = 2, sticky = 'nw')

        self.state_aa_4 = IntVar()
        self.name_aa_4 = StringVar()
        self.ip_aa_4 = StringVar()
        self.select_aa_4 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_4, onvalue=1 , offvalue=0)
        self.select_aa_4.grid(row = 3, column = 4,sticky = 'ne' )
        self.name_box_aa_4 = Entry(self.frame_selection, textvariable=self.name_aa_4)
        self.name_box_aa_4.grid(row = 3, column = 5, sticky = 'nw')
        self.ip_box_aa_4 = Entry(self.frame_selection, textvariable=self.ip_aa_4)
        self.ip_box_aa_4.grid(row = 3, column = 6, sticky = 'nw')

        self.state_aa_5 = IntVar()
        self.name_aa_5 = StringVar()
        self.ip_aa_5 = StringVar()
        self.select_aa_5 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_5, onvalue=1 , offvalue=0)
        self.select_aa_5.grid(row = 4, column = 0,sticky = 'ne' )
        self.name_box_aa_5 = Entry(self.frame_selection, textvariable=self.name_aa_5)
        self.name_box_aa_5.grid(row = 4, column = 1, sticky = 'nw')
        self.ip_box_aa_5 = Entry(self.frame_selection, textvariable=self.ip_aa_5)
        self.ip_box_aa_5.grid(row = 4, column = 2, sticky = 'nw')

        self.state_aa_6 = IntVar()
        self.name_aa_6 = StringVar()
        self.ip_aa_6 = StringVar()
        self.select_aa_6 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_6, onvalue=1 , offvalue=0)
        self.select_aa_6.grid(row = 4, column = 4,sticky = 'ne' )
        self.name_box_aa_6 = Entry(self.frame_selection, textvariable=self.name_aa_6)
        self.name_box_aa_6.grid(row = 4, column = 5, sticky = 'nw')
        self.ip_box_aa_6 = Entry(self.frame_selection, textvariable=self.ip_aa_6)
        self.ip_box_aa_6.grid(row = 4, column = 6, sticky = 'nw')

        self.state_aa_7 = IntVar()
        self.name_aa_7 = StringVar()
        self.ip_aa_7 = StringVar()
        self.select_aa_7 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_7, onvalue=1 , offvalue=0)
        self.select_aa_7.grid(row = 5, column = 0,sticky = 'ne' )
        self.name_box_aa_7 = Entry(self.frame_selection, textvariable=self.name_aa_7)
        self.name_box_aa_7.grid(row = 5, column = 1, sticky = 'nw')
        self.ip_box_aa_7 = Entry(self.frame_selection, textvariable=self.ip_aa_7)
        self.ip_box_aa_7.grid(row = 5, column = 2, sticky = 'nw')

        self.state_aa_8 = IntVar()
        self.name_aa_8 = StringVar()
        self.ip_aa_8 = StringVar()
        self.select_aa_8 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_8, onvalue=1 , offvalue=0)
        self.select_aa_8.grid(row = 5, column = 4,sticky = 'ne' )
        self.name_box_aa_8 = Entry(self.frame_selection, textvariable=self.name_aa_8)
        self.name_box_aa_8.grid(row = 5, column = 5, sticky = 'nw')
        self.ip_box_aa_8 = Entry(self.frame_selection, textvariable=self.ip_aa_8)
        self.ip_box_aa_8.grid(row = 5, column = 6, sticky = 'nw')

        self.state_aa_9 = IntVar()
        self.name_aa_9 = StringVar()
        self.ip_aa_9 = StringVar()
        self.select_aa_9 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_9, onvalue=1 , offvalue=0)
        self.select_aa_9.grid(row = 6, column = 0,sticky = 'ne' )
        self.name_box_aa_9 = Entry(self.frame_selection, textvariable=self.name_aa_9)
        self.name_box_aa_9.grid(row = 6, column = 1, sticky = 'nw')
        self.ip_box_aa_9 = Entry(self.frame_selection, textvariable=self.ip_aa_9)
        self.ip_box_aa_9.grid(row = 6, column = 2, sticky = 'nw')

        self.state_aa_10 = IntVar()
        self.name_aa_10 = StringVar()
        self.ip_aa_10 = StringVar()
        self.select_aa_10 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_10, onvalue=1 , offvalue=0)
        self.select_aa_10.grid(row = 6, column = 4,sticky = 'ne' )
        self.name_box_aa_10 = Entry(self.frame_selection, textvariable=self.name_aa_10)
        self.name_box_aa_10.grid(row = 6, column = 5, sticky = 'nw')
        self.ip_box_aa_10 = Entry(self.frame_selection, textvariable=self.ip_aa_10)
        self.ip_box_aa_10.grid(row = 6, column = 6, sticky = 'nw')

        self.state_aa_11 = IntVar()
        self.name_aa_11 = StringVar()
        self.ip_aa_11 = StringVar()
        self.select_aa_11 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_11, onvalue=1 , offvalue=0)
        self.select_aa_11.grid(row = 7, column = 0,sticky = 'ne' )
        self.name_box_aa_11 = Entry(self.frame_selection, textvariable=self.name_aa_11)
        self.name_box_aa_11.grid(row = 7, column = 1, sticky = 'nw')
        self.ip_box_aa_11 = Entry(self.frame_selection, textvariable=self.ip_aa_11)
        self.ip_box_aa_11.grid(row = 7, column = 2, sticky = 'nw')

        self.state_aa_12 = IntVar()
        self.name_aa_12 = StringVar()
        self.ip_aa_12 = StringVar()
        self.select_aa_12 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_12, onvalue=1 , offvalue=0)
        self.select_aa_12.grid(row =7, column = 4,sticky = 'ne' )
        self.name_box_aa_12 = Entry(self.frame_selection, textvariable=self.name_aa_12)
        self.name_box_aa_12.grid(row = 7, column = 5, sticky = 'nw')
        self.ip_box_aa_12 = Entry(self.frame_selection, textvariable=self.ip_aa_12)
        self.ip_box_aa_12.grid(row = 7, column = 6, sticky = 'nw')

        self.state_aa_13 = IntVar()
        self.name_aa_13 = StringVar()
        self.ip_aa_13 = StringVar()
        self.select_aa_13 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_13, onvalue=1 , offvalue=0)
        self.select_aa_13.grid(row = 8, column = 0,sticky = 'ne' )
        self.name_box_aa_13 = Entry(self.frame_selection, textvariable=self.name_aa_13)
        self.name_box_aa_13.grid(row = 8, column = 1, sticky = 'nw')
        self.ip_box_aa_13 = Entry(self.frame_selection, textvariable=self.ip_aa_13)
        self.ip_box_aa_13.grid(row = 8, column = 2, sticky = 'nw')

        self.state_aa_14 = IntVar()
        self.name_aa_14 = StringVar()
        self.ip_aa_14 = StringVar()
        self.select_aa_14 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_14, onvalue=1 , offvalue=0)
        self.select_aa_14.grid(row = 8, column = 4,sticky = 'ne' )
        self.name_box_aa_14 = Entry(self.frame_selection, textvariable=self.name_aa_14)
        self.name_box_aa_14.grid(row = 8, column = 5, sticky = 'nw')
        self.ip_box_aa_14 = Entry(self.frame_selection, textvariable=self.ip_aa_14)
        self.ip_box_aa_14.grid(row = 8, column = 6, sticky = 'nw')

        self.state_aa_15 = IntVar()
        self.name_aa_15 = StringVar()
        self.ip_aa_15 = StringVar()
        self.select_aa_15 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_15, onvalue=1 , offvalue=0)
        self.select_aa_15.grid(row = 9, column = 0,sticky = 'ne' )
        self.name_box_aa_15 = Entry(self.frame_selection, textvariable=self.name_aa_15)
        self.name_box_aa_15.grid(row = 9, column = 1, sticky = 'nw')
        self.ip_box_aa_15 = Entry(self.frame_selection, textvariable=self.ip_aa_15)
        self.ip_box_aa_15.grid(row = 9, column = 2, sticky = 'nw')

        self.state_aa_16 = IntVar()
        self.name_aa_16 = StringVar()
        self.ip_aa_16 = StringVar()
        self.select_aa_16 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_16, onvalue=1 , offvalue=0)
        self.select_aa_16.grid(row = 9, column = 4,sticky = 'ne' )
        self.name_box_aa_16 = Entry(self.frame_selection, textvariable=self.name_aa_16)
        self.name_box_aa_16.grid(row = 9, column = 5, sticky = 'nw')
        self.ip_box_aa_16 = Entry(self.frame_selection, textvariable=self.ip_aa_16)
        self.ip_box_aa_16.grid(row = 9, column = 6, sticky = 'nw')

        self.state_aa_17 = IntVar()
        self.name_aa_17 = StringVar()
        self.ip_aa_17 = StringVar()
        self.select_aa_17 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_17, onvalue=1 , offvalue=0)
        self.select_aa_17.grid(row = 10, column = 0,sticky = 'ne' )
        self.name_box_aa_17 = Entry(self.frame_selection, textvariable=self.name_aa_17)
        self.name_box_aa_17.grid(row = 10, column = 1, sticky = 'nw')
        self.ip_box_aa_17 = Entry(self.frame_selection, textvariable=self.ip_aa_17)
        self.ip_box_aa_17.grid(row = 10, column = 2, sticky = 'nw')

        self.state_aa_18 = IntVar()
        self.name_aa_18 = StringVar()
        self.ip_aa_18 = StringVar()
        self.select_aa_18 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_18, onvalue=1 , offvalue=0)
        self.select_aa_18.grid(row = 10, column = 4,sticky = 'ne' )
        self.name_box_aa_18 = Entry(self.frame_selection, textvariable=self.name_aa_18)
        self.name_box_aa_18.grid(row = 10, column = 5, sticky = 'nw')
        self.ip_box_aa_18 = Entry(self.frame_selection, textvariable=self.ip_aa_18)
        self.ip_box_aa_18.grid(row = 10, column = 6, sticky = 'nw')

        self.state_aa_19 = IntVar()
        self.name_aa_19 = StringVar()
        self.ip_aa_19 = StringVar()
        self.select_aa_19 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_19, onvalue=1 , offvalue=0)
        self.select_aa_19.grid(row = 11, column = 0,sticky = 'ne' )
        self.name_box_aa_19 = Entry(self.frame_selection, textvariable=self.name_aa_19)
        self.name_box_aa_19.grid(row = 11, column = 1, sticky = 'nw')
        self.ip_box_aa_19 = Entry(self.frame_selection, textvariable=self.ip_aa_19)
        self.ip_box_aa_19.grid(row = 11, column = 2, sticky = 'nw')

        self.state_aa_20 = IntVar()
        self.name_aa_20 = StringVar()
        self.ip_aa_20 = StringVar()
        self.select_aa_20 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_20, onvalue=1 , offvalue=0)
        self.select_aa_20.grid(row = 11, column = 4,sticky = 'ne' )
        self.name_box_aa_20 = Entry(self.frame_selection, textvariable=self.name_aa_20)
        self.name_box_aa_20.grid(row = 11, column = 5, sticky = 'nw')
        self.ip_box_aa_20 = Entry(self.frame_selection, textvariable=self.ip_aa_20)
        self.ip_box_aa_20.grid(row = 11, column = 6, sticky = 'nw')

        self.state_aa_21 = IntVar()
        self.name_aa_21 = StringVar()
        self.ip_aa_21 = StringVar()
        self.select_aa_21 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_21, onvalue=1 , offvalue=0)
        self.select_aa_21.grid(row = 12, column = 0,sticky = 'ne' )
        self.name_box_aa_21 = Entry(self.frame_selection, textvariable=self.name_aa_21)
        self.name_box_aa_21.grid(row = 12, column = 1, sticky = 'nw')
        self.ip_box_aa_21 = Entry(self.frame_selection, textvariable=self.ip_aa_21)
        self.ip_box_aa_21.grid(row = 12, column = 2, sticky = 'nw')

        self.state_aa_22 = IntVar()
        self.name_aa_22 = StringVar()
        self.ip_aa_22 = StringVar()
        self.select_aa_22 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_22, onvalue=1 , offvalue=0)
        self.select_aa_22.grid(row = 12, column = 4,sticky = 'ne' )
        self.name_box_aa_22 = Entry(self.frame_selection, textvariable=self.name_aa_22)
        self.name_box_aa_22.grid(row = 12, column = 5, sticky = 'nw')
        self.ip_box_aa_22 = Entry(self.frame_selection, textvariable=self.ip_aa_22)
        self.ip_box_aa_22.grid(row = 12, column = 6, sticky = 'nw')

        self.state_aa_23 = IntVar()
        self.name_aa_23 = StringVar()
        self.ip_aa_23 = StringVar()
        self.select_aa_23 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_23, onvalue=1 , offvalue=0)
        self.select_aa_23.grid(row = 13, column = 0,sticky = 'ne' )
        self.name_box_aa_23 = Entry(self.frame_selection, textvariable=self.name_aa_23)
        self.name_box_aa_23.grid(row = 13, column = 1, sticky = 'nw')
        self.ip_box_aa_23 = Entry(self.frame_selection, textvariable=self.ip_aa_23)
        self.ip_box_aa_23.grid(row = 13, column = 2, sticky = 'nw')

        self.state_aa_24 = IntVar()
        self.name_aa_24 = StringVar()
        self.ip_aa_24 = StringVar()
        self.select_aa_24 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_24, onvalue=1 , offvalue=0)
        self.select_aa_24.grid(row = 13, column = 4,sticky = 'ne' )
        self.name_box_aa_24 = Entry(self.frame_selection, textvariable=self.name_aa_24)
        self.name_box_aa_24.grid(row = 13, column = 5, sticky = 'nw')
        self.ip_box_aa_24 = Entry(self.frame_selection, textvariable=self.ip_aa_24)
        self.ip_box_aa_24.grid(row = 13, column = 6, sticky = 'nw')

        self.state_aa_25 = IntVar()
        self.name_aa_25 = StringVar()
        self.ip_aa_25 = StringVar()
        self.select_aa_25 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_25, onvalue=1 , offvalue=0)
        self.select_aa_25.grid(row = 14, column = 0,sticky = 'ne' )
        self.name_box_aa_25 = Entry(self.frame_selection, textvariable=self.name_aa_25)
        self.name_box_aa_25.grid(row = 14, column = 1, sticky = 'nw')
        self.ip_box_aa_25 = Entry(self.frame_selection, textvariable=self.ip_aa_25)
        self.ip_box_aa_25.grid(row = 14, column = 2, sticky = 'nw')

        self.state_aa_26 = IntVar()
        self.name_aa_26 = StringVar()
        self.ip_aa_26 = StringVar()
        self.select_aa_26 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_26, onvalue=1 , offvalue=0)
        self.select_aa_26.grid(row = 14, column = 4,sticky = 'ne' )
        self.name_box_aa_26 = Entry(self.frame_selection, textvariable=self.name_aa_26)
        self.name_box_aa_26.grid(row = 14, column = 5, sticky = 'nw')
        self.ip_box_aa_26 = Entry(self.frame_selection, textvariable=self.ip_aa_26)
        self.ip_box_aa_26.grid(row = 14, column = 6, sticky = 'nw')

        self.state_aa_27 = IntVar()
        self.name_aa_27 = StringVar()
        self.ip_aa_27 = StringVar()
        self.select_aa_27 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_27, onvalue=1 , offvalue=0)
        self.select_aa_27.grid(row = 15, column = 0,sticky = 'ne' )
        self.name_box_aa_27 = Entry(self.frame_selection, textvariable=self.name_aa_27)
        self.name_box_aa_27.grid(row = 15, column = 1, sticky = 'nw')
        self.ip_box_aa_27 = Entry(self.frame_selection, textvariable=self.ip_aa_27)
        self.ip_box_aa_27.grid(row = 15, column = 2, sticky = 'nw')

        self.state_aa_28 = IntVar()
        self.name_aa_28 = StringVar()
        self.ip_aa_28 = StringVar()
        self.select_aa_28 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_28, onvalue=1 , offvalue=0)
        self.select_aa_28.grid(row = 15, column = 4,sticky = 'ne' )
        self.name_box_aa_28 = Entry(self.frame_selection, textvariable=self.name_aa_28)
        self.name_box_aa_28.grid(row = 15, column = 5, sticky = 'nw')
        self.ip_box_aa_28 = Entry(self.frame_selection, textvariable=self.ip_aa_28)
        self.ip_box_aa_28.grid(row = 15, column = 6, sticky = 'nw')

        self.state_aa_29 = IntVar()
        self.name_aa_29 = StringVar()
        self.ip_aa_29 = StringVar()
        self.select_aa_29 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_29, onvalue=1 , offvalue=0)
        self.select_aa_29.grid(row = 16, column = 0,sticky = 'ne' )
        self.name_box_aa_29 = Entry(self.frame_selection, textvariable=self.name_aa_29)
        self.name_box_aa_29.grid(row = 16, column = 1, sticky = 'nw')
        self.ip_box_aa_29 = Entry(self.frame_selection, textvariable=self.ip_aa_29)
        self.ip_box_aa_29.grid(row = 16, column = 2, sticky = 'nw')

        self.state_aa_30 = IntVar()
        self.name_aa_30 = StringVar()
        self.ip_aa_30 = StringVar()
        self.select_aa_30 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_30, onvalue=1 , offvalue=0)
        self.select_aa_30.grid(row = 16, column = 4,sticky = 'ne' )
        self.name_box_aa_30 = Entry(self.frame_selection, textvariable=self.name_aa_30)
        self.name_box_aa_30.grid(row = 16, column = 5, sticky = 'nw')
        self.ip_box_aa_30 = Entry(self.frame_selection, textvariable=self.ip_aa_30)
        self.ip_box_aa_30.grid(row = 16, column = 6, sticky = 'nw')

        self.state_aa_31 = IntVar()
        self.name_aa_31 = StringVar()
        self.ip_aa_31 = StringVar()
        self.select_aa_31 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_31, onvalue=1 , offvalue=0)
        self.select_aa_31.grid(row = 17, column = 0,sticky = 'ne' )
        self.name_box_aa_31 = Entry(self.frame_selection, textvariable=self.name_aa_31)
        self.name_box_aa_31.grid(row = 17, column = 1, sticky = 'nw')
        self.ip_box_aa_31 = Entry(self.frame_selection, textvariable=self.ip_aa_31)
        self.ip_box_aa_31.grid(row = 17, column = 2, sticky = 'nw')

        self.state_aa_32 = IntVar()
        self.name_aa_32 = StringVar()
        self.ip_aa_32 = StringVar()
        self.select_aa_32 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_32, onvalue=1 , offvalue=0)
        self.select_aa_32.grid(row = 17, column = 4,sticky = 'ne' )
        self.name_box_aa_32 = Entry(self.frame_selection, textvariable=self.name_aa_32)
        self.name_box_aa_32.grid(row = 17, column = 5, sticky = 'nw')
        self.ip_box_aa_32 = Entry(self.frame_selection, textvariable=self.ip_aa_32)
        self.ip_box_aa_32.grid(row = 17, column = 6, sticky = 'nw')

        self.state_aa_33 = IntVar()
        self.name_aa_33 = StringVar()
        self.ip_aa_33 = StringVar()
        self.select_aa_33 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_33, onvalue=1 , offvalue=0)
        self.select_aa_33.grid(row = 18, column = 0,sticky = 'ne' )
        self.name_box_aa_33 = Entry(self.frame_selection, textvariable=self.name_aa_33)
        self.name_box_aa_33.grid(row = 18, column = 1, sticky = 'nw')
        self.ip_box_aa_33 = Entry(self.frame_selection, textvariable=self.ip_aa_33)
        self.ip_box_aa_33.grid(row = 18, column = 2, sticky = 'nw')

        self.state_aa_34 = IntVar()
        self.name_aa_34 = StringVar()
        self.ip_aa_34 = StringVar()
        self.select_aa_34 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_34, onvalue=1 , offvalue=0)
        self.select_aa_34.grid(row = 18, column = 4,sticky = 'ne' )
        self.name_box_aa_34 = Entry(self.frame_selection, textvariable=self.name_aa_34)
        self.name_box_aa_34.grid(row = 18, column = 5, sticky = 'nw')
        self.ip_box_aa_34 = Entry(self.frame_selection, textvariable=self.ip_aa_34)
        self.ip_box_aa_34.grid(row = 18, column = 6, sticky = 'nw')

        self.state_aa_35 = IntVar()
        self.name_aa_35 = StringVar()
        self.ip_aa_35 = StringVar()
        self.select_aa_35 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_35, onvalue=1 , offvalue=0)
        self.select_aa_35.grid(row = 19, column = 0,sticky = 'ne' )
        self.name_box_aa_35 = Entry(self.frame_selection, textvariable=self.name_aa_35)
        self.name_box_aa_35.grid(row = 19, column = 1, sticky = 'nw')
        self.ip_box_aa_35 = Entry(self.frame_selection, textvariable=self.ip_aa_35)
        self.ip_box_aa_35.grid(row = 19, column = 2, sticky = 'nw')

        self.state_aa_36 = IntVar()
        self.name_aa_36 = StringVar()
        self.ip_aa_36 = StringVar()
        self.select_aa_36 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_36, onvalue=1 , offvalue=0)
        self.select_aa_36.grid(row = 19, column = 4,sticky = 'ne' )
        self.name_box_aa_36 = Entry(self.frame_selection, textvariable=self.name_aa_36)
        self.name_box_aa_36.grid(row = 19, column = 5, sticky = 'nw')
        self.ip_box_aa_36 = Entry(self.frame_selection, textvariable=self.ip_aa_36)
        self.ip_box_aa_36.grid(row = 19, column = 6, sticky = 'nw')

        self.state_aa_37 = IntVar()
        self.name_aa_37 = StringVar()
        self.ip_aa_37 = StringVar()
        self.select_aa_37 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_37, onvalue=1 , offvalue=0)
        self.select_aa_37.grid(row = 20, column = 0,sticky = 'ne' )
        self.name_box_aa_37 = Entry(self.frame_selection, textvariable=self.name_aa_37)
        self.name_box_aa_37.grid(row = 20, column = 1, sticky = 'nw')
        self.ip_box_aa_37 = Entry(self.frame_selection, textvariable=self.ip_aa_37)
        self.ip_box_aa_37.grid(row = 20, column = 2, sticky = 'nw')

        self.state_aa_38 = IntVar()
        self.name_aa_38 = StringVar()
        self.ip_aa_38 = StringVar()
        self.select_aa_38 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_38, onvalue=1 , offvalue=0)
        self.select_aa_38.grid(row = 20, column = 4,sticky = 'ne' )
        self.name_box_aa_38 = Entry(self.frame_selection, textvariable=self.name_aa_38)
        self.name_box_aa_38.grid(row = 20, column = 5, sticky = 'nw')
        self.ip_box_aa_38 = Entry(self.frame_selection, textvariable=self.ip_aa_38)
        self.ip_box_aa_38.grid(row = 20, column = 6, sticky = 'nw')

        self.state_aa_39 = IntVar()
        self.name_aa_39 = StringVar()
        self.ip_aa_39 = StringVar()
        self.select_aa_39 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_39, onvalue=1 , offvalue=0)
        self.select_aa_39.grid(row = 21, column = 0,sticky = 'ne' )
        self.name_box_aa_39 = Entry(self.frame_selection, textvariable=self.name_aa_39)
        self.name_box_aa_39.grid(row = 21, column = 1, sticky = 'nw')
        self.ip_box_aa_39 = Entry(self.frame_selection, textvariable=self.ip_aa_39)
        self.ip_box_aa_39.grid(row = 21, column = 2, sticky = 'nw')

        self.state_aa_40 = IntVar()
        self.name_aa_40 = StringVar()
        self.ip_aa_40 = StringVar()
        self.select_aa_40 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_40, onvalue=1 , offvalue=0)
        self.select_aa_40.grid(row = 21, column = 4,sticky = 'ne' )
        self.name_box_aa_40 = Entry(self.frame_selection, textvariable=self.name_aa_40)
        self.name_box_aa_40.grid(row = 21, column = 5, sticky = 'nw')
        self.ip_box_aa_40 = Entry(self.frame_selection, textvariable=self.ip_aa_40)
        self.ip_box_aa_40.grid(row = 21, column = 6, sticky = 'nw')


        self.state_aa_41 = IntVar()
        self.name_aa_41 = StringVar()
        self.ip_aa_41 = StringVar()
        self.select_aa_41 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_40, onvalue=1 , offvalue=0)
        self.select_aa_41.grid(row = 2, column = 7,sticky = 'ne' )
        self.name_box_aa_41 = Entry(self.frame_selection, textvariable=self.name_aa_41)
        self.name_box_aa_41.grid(row = 2, column = 8, sticky = 'nw')
        self.ip_box_aa_41 = Entry(self.frame_selection, textvariable=self.ip_aa_41)
        self.ip_box_aa_41.grid(row = 2, column = 9, sticky = 'nw')


        self.state_aa_42 = IntVar()
        self.name_aa_42 = StringVar()
        self.ip_aa_42 = StringVar()
        self.select_aa_42 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_42, onvalue=1 , offvalue=0)
        self.select_aa_42.grid(row = 3, column = 7,sticky = 'ne' )
        self.name_box_aa_42 = Entry(self.frame_selection, textvariable=self.name_aa_42)
        self.name_box_aa_42.grid(row = 3, column = 8, sticky = 'nw')
        self.ip_box_aa_42 = Entry(self.frame_selection, textvariable=self.ip_aa_42)
        self.ip_box_aa_42.grid(row = 3, column = 9, sticky = 'nw')

        self.state_aa_43 = IntVar()
        self.name_aa_43 = StringVar()
        self.ip_aa_43 = StringVar()
        self.select_aa_43 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_43, onvalue=1 , offvalue=0)
        self.select_aa_43.grid(row = 4, column = 7,sticky = 'ne' )
        self.name_box_aa_43 = Entry(self.frame_selection, textvariable=self.name_aa_43)
        self.name_box_aa_43.grid(row = 4, column = 8, sticky = 'nw')
        self.ip_box_aa_43 = Entry(self.frame_selection, textvariable=self.ip_aa_43)
        self.ip_box_aa_43.grid(row = 4, column = 9, sticky = 'nw')

        self.state_aa_44 = IntVar()
        self.name_aa_44 = StringVar()
        self.ip_aa_44 = StringVar()
        self.select_aa_44 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_44, onvalue=1 , offvalue=0)
        self.select_aa_44.grid(row = 5, column = 7,sticky = 'ne' )
        self.name_box_aa_44 = Entry(self.frame_selection, textvariable=self.name_aa_44)
        self.name_box_aa_44.grid(row = 5, column = 8, sticky = 'nw')
        self.ip_box_aa_44 = Entry(self.frame_selection, textvariable=self.ip_aa_44)
        self.ip_box_aa_44.grid(row = 5, column = 9, sticky = 'nw')

        self.state_aa_45 = IntVar()
        self.name_aa_45 = StringVar()
        self.ip_aa_45 = StringVar()
        self.select_aa_45 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_45, onvalue=1 , offvalue=0)
        self.select_aa_45.grid(row = 6, column = 7,sticky = 'ne' )
        self.name_box_aa_45 = Entry(self.frame_selection, textvariable=self.name_aa_45)
        self.name_box_aa_45.grid(row = 6, column = 8, sticky = 'nw')
        self.ip_box_aa_45 = Entry(self.frame_selection, textvariable=self.ip_aa_45)
        self.ip_box_aa_45.grid(row = 6, column = 9, sticky = 'nw')

        self.state_aa_46 = IntVar()
        self.name_aa_46 = StringVar()
        self.ip_aa_46 = StringVar()
        self.select_aa_46 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_46, onvalue=1 , offvalue=0)
        self.select_aa_46.grid(row = 7, column = 7,sticky = 'ne' )
        self.name_box_aa_46 = Entry(self.frame_selection, textvariable=self.name_aa_46)
        self.name_box_aa_46.grid(row = 7, column = 8, sticky = 'nw')
        self.ip_box_aa_46 = Entry(self.frame_selection, textvariable=self.ip_aa_46)
        self.ip_box_aa_46.grid(row = 7, column = 9, sticky = 'nw')

        self.state_aa_47 = IntVar()
        self.name_aa_47 = StringVar()
        self.ip_aa_47 = StringVar()
        self.select_aa_47 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_47, onvalue=1 , offvalue=0)
        self.select_aa_47.grid(row = 8, column = 7,sticky = 'ne' )
        self.name_box_aa_47 = Entry(self.frame_selection, textvariable=self.name_aa_47)
        self.name_box_aa_47.grid(row = 8, column = 8, sticky = 'nw')
        self.ip_box_aa_47 = Entry(self.frame_selection, textvariable=self.ip_aa_47)
        self.ip_box_aa_47.grid(row = 8, column = 9, sticky = 'nw')

        self.state_aa_48 = IntVar()
        self.name_aa_48 = StringVar()
        self.ip_aa_48 = StringVar()
        self.select_aa_48 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_48, onvalue=1 , offvalue=0)
        self.select_aa_48.grid(row = 9, column = 7,sticky = 'ne' )
        self.name_box_aa_48 = Entry(self.frame_selection, textvariable=self.name_aa_48)
        self.name_box_aa_48.grid(row = 9, column = 8, sticky = 'nw')
        self.ip_box_aa_48 = Entry(self.frame_selection, textvariable=self.ip_aa_48)
        self.ip_box_aa_48.grid(row = 9, column = 9, sticky = 'nw')

        self.state_aa_49 = IntVar()
        self.name_aa_49 = StringVar()
        self.ip_aa_49 = StringVar()
        self.select_aa_49 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_49, onvalue=1 , offvalue=0)
        self.select_aa_49.grid(row = 10, column = 7,sticky = 'ne' )
        self.name_box_aa_49 = Entry(self.frame_selection, textvariable=self.name_aa_49)
        self.name_box_aa_49.grid(row = 10, column = 8, sticky = 'nw')
        self.ip_box_aa_49 = Entry(self.frame_selection, textvariable=self.ip_aa_49)
        self.ip_box_aa_49.grid(row = 10, column = 9, sticky = 'nw')

        self.state_aa_50 = IntVar()
        self.name_aa_50 = StringVar()
        self.ip_aa_50 = StringVar()
        self.select_aa_50 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_50, onvalue=1 , offvalue=0)
        self.select_aa_50.grid(row = 11, column = 7,sticky = 'ne' )
        self.name_box_aa_50 = Entry(self.frame_selection, textvariable=self.name_aa_50)
        self.name_box_aa_50.grid(row = 11, column = 8, sticky = 'nw')
        self.ip_box_aa_50 = Entry(self.frame_selection, textvariable=self.ip_aa_50)
        self.ip_box_aa_50.grid(row = 11, column = 9, sticky = 'nw')

        self.state_aa_51 = IntVar()
        self.name_aa_51 = StringVar()
        self.ip_aa_51 = StringVar()
        self.select_aa_51 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_51, onvalue=1 , offvalue=0)
        self.select_aa_51.grid(row = 12, column = 7,sticky = 'ne' )
        self.name_box_aa_51 = Entry(self.frame_selection, textvariable=self.name_aa_51)
        self.name_box_aa_51.grid(row = 12, column = 8, sticky = 'nw')
        self.ip_box_aa_51 = Entry(self.frame_selection, textvariable=self.ip_aa_51)
        self.ip_box_aa_51.grid(row = 12, column = 9, sticky = 'nw')

        self.state_aa_52 = IntVar()
        self.name_aa_52 = StringVar()
        self.ip_aa_52 = StringVar()
        self.select_aa_52 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_52, onvalue=1 , offvalue=0)
        self.select_aa_52.grid(row = 13, column = 7,sticky = 'ne' )
        self.name_box_aa_52 = Entry(self.frame_selection, textvariable=self.name_aa_52)
        self.name_box_aa_52.grid(row = 13, column = 8, sticky = 'nw')
        self.ip_box_aa_52 = Entry(self.frame_selection, textvariable=self.ip_aa_52)
        self.ip_box_aa_52.grid(row = 13, column = 9, sticky = 'nw')

        self.state_aa_53 = IntVar()
        self.name_aa_53 = StringVar()
        self.ip_aa_53 = StringVar()
        self.select_aa_53 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_53, onvalue=1 , offvalue=0)
        self.select_aa_53.grid(row =14, column = 7,sticky = 'ne' )
        self.name_box_aa_53 = Entry(self.frame_selection, textvariable=self.name_aa_53)
        self.name_box_aa_53.grid(row = 14, column = 8, sticky = 'nw')
        self.ip_box_aa_53 = Entry(self.frame_selection, textvariable=self.ip_aa_53)
        self.ip_box_aa_53.grid(row = 14, column = 9, sticky = 'nw')

        self.state_aa_54 = IntVar()
        self.name_aa_54 = StringVar()
        self.ip_aa_54 = StringVar()
        self.select_aa_54 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_54, onvalue=1 , offvalue=0)
        self.select_aa_54.grid(row = 15, column = 7,sticky = 'ne' )
        self.name_box_aa_54 = Entry(self.frame_selection, textvariable=self.name_aa_54)
        self.name_box_aa_54.grid(row = 15, column = 8, sticky = 'nw')
        self.ip_box_aa_54 = Entry(self.frame_selection, textvariable=self.ip_aa_54)
        self.ip_box_aa_54.grid(row = 15, column = 9, sticky = 'nw')

        self.state_aa_55 = IntVar()
        self.name_aa_55 = StringVar()
        self.ip_aa_55 = StringVar()
        self.select_aa_55 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_55, onvalue=1 , offvalue=0)
        self.select_aa_55.grid(row = 16, column = 7,sticky = 'ne' )
        self.name_box_aa_55 = Entry(self.frame_selection, textvariable=self.name_aa_55)
        self.name_box_aa_55.grid(row = 16, column = 8, sticky = 'nw')
        self.ip_box_aa_55 = Entry(self.frame_selection, textvariable=self.ip_aa_55)
        self.ip_box_aa_55.grid(row = 16, column = 9, sticky = 'nw')

        self.state_aa_56 = IntVar()
        self.name_aa_56 = StringVar()
        self.ip_aa_56 = StringVar()
        self.select_aa_56 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_56, onvalue=1 , offvalue=0)
        self.select_aa_56.grid(row = 17, column = 7,sticky = 'ne' )
        self.name_box_aa_56 = Entry(self.frame_selection, textvariable=self.name_aa_56)
        self.name_box_aa_56.grid(row = 17, column = 8, sticky = 'nw')
        self.ip_box_aa_56 = Entry(self.frame_selection, textvariable=self.ip_aa_56)
        self.ip_box_aa_56.grid(row = 17, column = 9, sticky = 'nw')

        self.state_aa_57 = IntVar()
        self.name_aa_57 = StringVar()
        self.ip_aa_57 = StringVar()
        self.select_aa_57 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_57, onvalue=1 , offvalue=0)
        self.select_aa_57.grid(row = 18, column = 7,sticky = 'ne' )
        self.name_box_aa_57 = Entry(self.frame_selection, textvariable=self.name_aa_57)
        self.name_box_aa_57.grid(row = 18, column = 8, sticky = 'nw')
        self.ip_box_aa_57 = Entry(self.frame_selection, textvariable=self.ip_aa_57)
        self.ip_box_aa_57.grid(row = 18, column = 9, sticky = 'nw')

        self.state_aa_58 = IntVar()
        self.name_aa_58 = StringVar()
        self.ip_aa_58 = StringVar()
        self.select_aa_58 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_58, onvalue=1 , offvalue=0)
        self.select_aa_58.grid(row = 19, column = 7,sticky = 'ne' )
        self.name_box_aa_58 = Entry(self.frame_selection, textvariable=self.name_aa_58)
        self.name_box_aa_58.grid(row = 19, column = 8, sticky = 'nw')
        self.ip_box_aa_58 = Entry(self.frame_selection, textvariable=self.ip_aa_58)
        self.ip_box_aa_58.grid(row = 19, column = 9, sticky = 'nw')

        self.state_aa_59 = IntVar()
        self.name_aa_59 = StringVar()
        self.ip_aa_59 = StringVar()
        self.select_aa_59 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_59, onvalue=1 , offvalue=0)
        self.select_aa_59.grid(row = 20, column = 7,sticky = 'ne' )
        self.name_box_aa_59 = Entry(self.frame_selection, textvariable=self.name_aa_59)
        self.name_box_aa_59.grid(row = 20, column = 8, sticky = 'nw')
        self.ip_box_aa_59 = Entry(self.frame_selection, textvariable=self.ip_aa_59)
        self.ip_box_aa_59.grid(row = 20, column = 9, sticky = 'nw')


        self.state_aa_60 = IntVar()
        self.name_aa_60 = StringVar()
        self.ip_aa_60 = StringVar()
        self.select_aa_60 = ttk.Checkbutton(self.frame_selection, text='Enable', variable=self.state_aa_60, onvalue=1 , offvalue=0)
        self.select_aa_60.grid(row = 21, column = 7,sticky = 'ne' )
        self.name_box_aa_60 = Entry(self.frame_selection, textvariable=self.name_aa_60)
        self.name_box_aa_60.grid(row = 21, column = 8, sticky = 'nw')
        self.ip_box_aa_60 = Entry(self.frame_selection, textvariable=self.ip_aa_60)
        self.ip_box_aa_60.grid(row = 21, column = 9, sticky = 'nw')



        # This command reads the server_list.csv file and populates all 40 entry selections
        self.populate_config_columns()

        # This command deletes entry selections if there is no IP address assigned
        self.delete_unpopulated_rows()

        # This is a separator between the entry selection columns
        ttk.Separator(self.frame_selection, orient=VERTICAL).grid(column=3, row=2, rowspan=21, sticky='ns')



        # this frame contains the Pull AA confs files button
        self.frame_execute_buttons = ttk.Frame(self)
        self.frame_execute_buttons.pack()

        self.pull_conf_button = ttk.Button(self.frame_execute_buttons, text = 'Pull files from Servers', command =lambda: self.pull_files())
        self.pull_conf_button.grid(row=0, column=0, sticky = 'w', padx =20, pady=20, ipadx=20, ipady=20 )

        self.open_webpages_button = ttk.Button(self.frame_execute_buttons, text = 'Open Webpage to Servers', command =lambda: self.openWebTabs())
        self.open_webpages_button.grid(row=0, column=1, sticky = 'w', padx =20, pady=20, ipadx=20, ipady=20)

        self.push_files_button = ttk.Button(self.frame_execute_buttons, text = 'Push files to Servers', command =lambda: self.push_file_to_servers())
        self.push_files_button.grid(row=0, column=2, sticky = 'w', padx =20, pady=20, ipadx=20, ipady=20)


        #This pane contains the text log box and save button
        self.frame_textwork = ttk.Frame(self)
        self.frame_textwork.pack()

        ttk.Label(self.frame_textwork, text = 'Console Output: ').grid(row = 1, column = 0, sticky = 'nw', padx = 5)



        self.exe_content = StringVar()
        self.exe_content = ''

        self.execute_text = Text(self.frame_textwork, width = 83, height = 10, wrap='none')


        xscroll1 = ttk.Scrollbar(self.frame_textwork, orient = HORIZONTAL, command = self.execute_text.xview)
        yscroll1 = ttk.Scrollbar(self.frame_textwork, orient = VERTICAL, command = self.execute_text.yview)
        self.execute_text.config(xscrollcommand = xscroll1.set, yscrollcommand = yscroll1.set)


        self.execute_text.grid(row = 2, column = 0, sticky = 'nw' , padx = 5)
        xscroll1.grid(row = 3, column = 0, sticky = 'ew')
        yscroll1.grid(row = 2, column = 1, sticky = 'ns')


        # This frame contains the code version as pulled from the main class variable
        self.frame_version = ttk.Frame(self)
        self.frame_version.pack()
        ttk.Label(self.frame_version, text = MainFrame.code_version).grid(row = 1, column = 0, sticky = 'nw', pady = 35)

    def select_directory(self):
        """This function defines the directory selection button's actions"""
        self.dir_entry.delete(0, END)

        selected_dir =  filedialog.askdirectory(initialdir = "/",title = "Select a Directory to Pull files")
        self.dir_entry.insert("insert", selected_dir)

    def select_push_file(self):
        """This function defines a file to push button's actions"""
        self.push_file_entry.delete(0, END)

        select_file = filedialog.askopenfilename(initialdir= "/", title="Select a File to Push to Adapters")
        self.push_file_entry.insert("insert", select_file)

    def push_file_to_servers(self):
        file_to_push = self.push_file_entry.get()
        target_for_push = self.push_location_entry.get()
        entered_username = self.server_username.get()
        entered_password = self.server_password.get()


        #Determine the selected entry boxes
        selected_ips = self.determine_selected_ips_v2()

        if entered_username == '':
            self.insert_message("Error: please enter a username and try again")
            return

        if entered_password == '':
            self.insert_message("Error: please enter a password and try again")
            return

        if file_to_push == '':
            self.insert_message("Error: please select a file to push to servers")
            return
        else:
            self.insert_message("File "+ str(file_to_push) + " selected for transfer")

        if target_for_push == '':
            self.insert_message("Error: please select a target location on servers")
            return
        else:
            self.insert_message("Push location set to " + str(target_for_push))


        # This requires that an entry box with an IP address is required. If not supplied, stop the function
        if len(selected_ips) == 0:
            self.insert_message("Error: Please select at least one server IP Address to transfer. ")
            return
        else:
            self.insert_message("IP Addresses selected for push:")
            for ip in selected_ips:
                self.insert_message_no_timestamp("\t\t" + str(ip))



        for ip in selected_ips:
            try:
                self.insert_message("Pushing files to " + str(ip))
                # Create the SSH session
                ssh = self.createSSHClient(server=ip, port=22, user=entered_username, password=entered_password)
                # Create the transport steam
                scp = SCPClient(ssh.get_transport())
                # Start the ssh session
                sftp = ssh.open_sftp()

                transfer_filename = os.path.basename(file_to_push)
                if target_for_push.endswith("/"):
                    full_target = target_for_push + transfer_filename
                else:
                    full_target = target_for_push + "/" + transfer_filename


                scp.put(file_to_push, full_target)


                self.insert_message("Process complete")
                ssh.close()
            except TimeoutError:
                # Timeouts can happen and will be printed to the output box if a retry is desired
                self.insert_message("Timeout error encountered for IP " + str(ip))

            except Exception as e:
                self.insert_message("Error encountered: " + str(e))

    def pull_files(self):
        """This function defines the pull file button's actions."""

        pull_path = self.file_to_pull.get()
        entered_username = self.server_username.get()
        entered_password = self.server_password.get()

        if pull_path == '':
            self.insert_message("Error: No file selected to pull")
            return

        if entered_username == '':
            self.insert_message("Error: please enter a username and try again")
            return

        if entered_password == '':
            self.insert_message("Error: please enter a password and try again")
            return


        #Determine the selected entry boxes
        selected_ips = self.determine_selected_ips_v2()

        #Determine the selected transfer directory from the entry box
        selected_dir = self.transfer_target.get()

        if selected_dir == '':
            self.insert_message("Error: please select a directory for pulled files ")
            return
        else:
            self.insert_message("Directory " + selected_dir + " selected to transfer files ")



        # This requires that an entry box with an IP address is required. If not supplied, stop the function
        if len(selected_ips) == 0:
            self.insert_message("Error: Please select at least one IP Address to transfer. ")
            return
        else:
            self.insert_message("IP Addresses selected for transfer:")
            for ip in selected_ips:
                self.insert_message_no_timestamp("\t\t" + str(ip))

        pull_dir = str(self.get_file_path(pull_path)) + "/"
        print(pull_dir)
        pull_name = str(self.get_file_name(pull_path))

        current_datetime = datetime.datetime.now().replace(microsecond=0).isoformat().replace(":", "")

        # Being the for loop with the list of IPs
        for ip in selected_ips:
            try:
                self.insert_message("Transferring files from " + str(ip))
                # Create the SSH session
                ssh = self.createSSHClient(server=ip, port=22, user=entered_username, password=entered_password)
                # Create the transport steam
                print("step 1 connection complete")
                scp = SCPClient(ssh.get_transport())
                # Start the ssh session
                print("Step 2")
                sftp = ssh.open_sftp()
                # List all files in the config file path
                print("Step 3")
                files = sftp.listdir(path=pull_dir)
                print(files)
                # If no files found at the targeted IP address return this message
                if len(files) == 0:
                    self.insert_message("No files found to transfer. Please check the IP address")
                # For files that match the advatar.conf file name, transfer them
                for file in files:
                    if file == pull_name:
                        file_remote_path = pull_path
                        self.insert_message("transferring file " + str(file))
                        scp.get(remote_path=file_remote_path, local_path=selected_dir)
                        downloaded_file = selected_dir + '/' + file
                        ending = self.ip_selection_naming_v2(ip)
                        try:
                            new_file = downloaded_file + '_' + ending + '_' + current_datetime
                            os.rename(downloaded_file,new_file)
                        except FileExistsError:
                            newer_file = downloaded_file + '_' + ending + '_' + current_datetime + 'N'
                            os.rename(downloaded_file,newer_file)

                self.insert_message("Process complete")
                ssh.close()
            except TimeoutError:
                # Timeouts can happen and will be printed to the output box if a retry is desired
                self.insert_message("Timeout error encountered for IP " + str(ip))
            except FileNotFoundError:
                self.insert_message("Error: File not found for IP")


    def openWebTabs(self):
        selected = self.determine_selected_ips_v2()

        chrome_path = r"C:\Program Files\Google\Chrome\Application\chrome.exe"
        firefox_path = r"C:\Program Files\Mozilla Firefox\firefox.exe"
        webbrowser.register('chrome', None, webbrowser.BackgroundBrowser(chrome_path))
        webbrowser.register('firefox', None, webbrowser.BackgroundBrowser(firefox_path))

        if os.path.isfile(chrome_path):
            for ip in selected:
                webbrowser.get(using='chrome').open_new_tab(ip)
        elif os.path.isfile(firefox_path):
            for ip in selected:
                webbrowser.get(using='firefox').open_new_tab(ip)
        else:
            for ip in selected:
                webbrowser.open_new_tab(ip)


    def createSSHClient(self, server, port, user, password):
        """This function creates the SSH client"""
        client = paramiko.SSHClient()
        client.load_system_host_keys()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        client.connect(hostname=server, port=port, username=user, password=password)
        client.get_transport().set_keepalive(2)
        # Larger window size for faster transfer speeds
        client.get_transport().window_size = 3 * 1024 * 1024

        return client


    def populate_config_columns(self):
        """This function populates the targeted AA list"""

        file_length = self.determine_csv_length()
        # If there are more than 40 lines, stop the import process. Have less than this
        if file_length > 60:
            messagebox.showerror(title="Server List Error", message="The server_list.csv file has too many rows. "
                                                                "60 is the maximum allowed. "
                                                                "Please adjust the file and reopen "
                                                                "the program to reload the list.")
            return

        # for the amount of lines in the server_list.csv file, read the lines and insert them into the entry boxes
        i = 1
        current_path = os.getcwd()
        default_csv_filename = 'server_list.csv'
        default_filepath = current_path + "/" + default_csv_filename
        if os.path.exists(default_filepath):
            with open(default_filepath, mode='r') as file:
                config_file = csv.reader(file)
                for lines in config_file:
                    next_name_obj = "name_box_aa_" + str(i)
                    next_ip_obj = "ip_box_aa_" + str(i)
                    name_command = 'self.' + next_name_obj + '.insert("insert", "' + str(lines[0]) + '")'
                    exec(name_command)
                    ip_command = 'self.' + next_ip_obj + '.insert("insert", "' + str(lines[1]) + '")'
                    exec(ip_command)
                    i += 1
    def delete_unpopulated_rows(self):
        """This function deletes server entry checkboxes and entry boxes if the IP address is empty after csv file import"""

        # List of strings to help construct exec commands
        list_of_ip_obj = ["_aa_1", "_aa_2", "_aa_3", "_aa_4",
                          "_aa_5", "_aa_6", "_aa_7", "_aa_8",
                          "_aa_9", "_aa_10", "_aa_11", "_aa_12",
                          "_aa_13", "_aa_14", "_aa_15", "_aa_16",
                          "_aa_17", "_aa_18", "_aa_19", "_aa_20",
                          "_aa_21", "_aa_22", "_aa_23", "_aa_24",
                          "_aa_25", "_aa_26", "_aa_27", "_aa_28",
                          "_aa_29", "_aa_30", "_aa_31", "_aa_32",
                          "_aa_33", "_aa_34", "_aa_35", "_aa_36",
                          "_aa_37", "_aa_38", "_aa_39", "_aa_40",
                          "_aa_41", "_aa_42", "_aa_43", "_aa_44",
                          "_aa_45", "_aa_46", "_aa_47", "_aa_48",
                          "_aa_49", "_aa_50", "_aa_51", "_aa_52",
                          "_aa_53", "_aa_54", "_aa_55", "_aa_56",
                          "_aa_57", "_aa_58", "_aa_59", "_aa_60"]


        # for all the ip_box entry boxes, if empty, construct a exec command to destroy the tkinter object
        for entry in list_of_ip_obj:
            get_command = "if " + "self.ip_box" +  entry + ".get() == '':\n" + "\t" + "self.ip_box" \
                          + entry + ".destroy()\n" + "\t" + "self.name_box" + entry + ".destroy()\n" \
                          + "\t" + "self.select" + entry + ".destroy()"


            exec(get_command)

    def select_all_button(self):
        """This function checks the amount of checkbuttons that exist, and them sets that amount of checkbox states
        to 1 meaning enabled """
        amount_intvars = 0
        i = 1
        for n in self.frame_selection.winfo_children():
            if re.search(r'checkbutton', str(n)):
                amount_intvars += 1
        #
        while i < (amount_intvars + 1):
            set_command = "self.state_aa_" + str(i) + ".set(1)"
            exec(set_command)
            i += 1

    def deselect_all_button(self):
        """This function checks the amount of checkbuttons that exist, and them sets that amount of checkbox states
        to 0 meaning disabled """

        amount_intvars = 0
        i = 1
        for n in self.frame_selection.winfo_children():
            if re.search(r'checkbutton', str(n)):
                amount_intvars += 1
        #
        while i < (amount_intvars + 1):
            set_command = "self.state_aa_" + str(i) + ".set(0)"
            exec(set_command)
            i += 1

    def update_date(self):
        """This function is used by the calendar entry box and calendar widget to update the date """
        self.date_entry.delete(0, END)
        date = self.cal.get_date()
        self.date_entry.insert(0, date)


    def determine_selected_ips_v2(self):
        """Testing function to check the state of each checkbox to determine which IPs are listed"""

        # Declare a list of selected IP address
        list_of_selected = []

        #Declare a set for selected IPs.
        selected_set = set()

        for i in range(1, 61):
            checkbox = "self.state_aa_" + str(i) + ".get()"
            ipbox = "self.ip_box_aa_" + str(i) + ".get()"

            if eval(checkbox) == 1:
                append_item = eval(ipbox)
                list_of_selected.append(append_item)

        # Update the set to the list of selected IPs. A set is used as Python sets do not allow duplicates.
        # This means that a unique list will be returned
        selected_set = set(list_of_selected)

        return selected_set


    def ip_selection_naming_v2(self, ip):

        name = ''

        for i in range(1, 60):
            ip_selection = "self.ip_aa_" + str(i) + ".get()"
            name_selection = "self.name_aa_" + str(i) + ".get()"

            if ip == eval(ip_selection):
                name = eval(name_selection)

        if name == '':
            name = 'unknown'

        return name


    def get_file_path(self, file_path):
        path = ''
        file_path_components = file_path.split('/')
        path_list = file_path_components
        for i in path_list[:-1]:

         path = path + i + '/'
        return path
    def get_file_name(self, file_path):
        file_path_components = file_path.split('/')
        file_name_and_extension = file_path_components[-1]
        return file_name_and_extension

    def insert_message(self, message):
        """This function helps reduce the amount of code needed to insert a log with time formatting."""
        raw_timenow = datetime.datetime.now()
        format_timenow = raw_timenow.strftime("%Y-%m-%d %H:%M:%S")
        self.execute_text.insert(END, (format_timenow + ": " + message + "\n"))
        self.execute_text.see("end")

    def insert_message_no_timestamp(self, message):
        """This function helps reduce the amount of code needed to insert a log with time formatting. Useful for lists
        of messages to make it appear they have the same exact time value when time is not needed. """

        self.execute_text.insert(END, (message + "\n"))
        self.execute_text.see("end")


    def determine_csv_length(self):
        """This function determines the length of the server_list.csv file"""
        csv_length = 0
        current_path = os.getcwd()
        default_csv_filename = 'server_list.csv'
        default_filepath = current_path + "/" + default_csv_filename
        if os.path.exists(default_filepath):
            with open(default_filepath, mode='r') as file:
                config_file = csv.reader(file)
                for lines in config_file:
                    csv_length += 1

        return csv_length


# Tkinter App stuff
if __name__ == '__main__':
    app = MainFrame()
    app.mainloop()


# build instructions for py-installer: pyinstaller --windowed  --add-data "pulley.ico;." --name PulliGUI --onefile app.py

```

