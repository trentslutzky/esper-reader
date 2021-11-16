#!/bin/python3.9
import curses
import vlc
from time import sleep
from tinytag import TinyTag
import os
import sqlite3 as sql
import argparse

parser = argparse.ArgumentParser(description='Plays an audiobook (audio file) and saves where you were when closing.')
parser.add_argument('file_path', metavar='<audio file>', type=str, help='file to open')
parser.add_argument('-r', action='store_true', dest='restart', default=False, help='start from the beginning of the file')
args = parser.parse_args()

restart = False

file_path = args.file_path

# initialize the screen objet 
w = curses.initscr()

# color
curses.start_color()

# make curses happy
curses.noecho()
curses.cbreak()
w.keypad(True)

# hide the cursor
curses.curs_set(0)
curses.mousemask(1)

# keys
KEY_SPACE = ord(' ')
KEY_P = ord('p')
KEY_Q = ord('q')

curses.init_pair(1, curses.COLOR_WHITE, curses.COLOR_BLACK)
curses.init_pair(2, curses.COLOR_CYAN, curses.COLOR_BLACK)
curses.init_pair(3, curses.COLOR_RED, curses.COLOR_BLACK)

def generate_progress_bar(current,total):
    pass

def millis_to_timestr(millis):
        seconds=(millis/1000)%60
        seconds = str(int(seconds))
        seconds = seconds.zfill(2)
        minutes=(millis/(1000*60))%60
        minutes = str(int(minutes))
        minutes = minutes.zfill(2)
        hours=(millis/(1000*60*60))%24
        hours = str(int(hours))
        hours = hours.zfill(2)

        time_str = hours+':'+minutes+':'+seconds

        return time_str


def make_table(cur):
    cur.execute("""
        CREATE TABLE books
        (file, millis)""")

def main(w):
    con = sql.connect('books.db')
    cur = con.cursor()

    f = vlc.MediaPlayer(file_path)
    
    try:
        make_table(cur)
    except:
        pass

    millis = cur.execute(f"SELECT millis FROM books where file = '{file_path}'").fetchone()
    if millis:
        millis = millis[0]
    else:
        cur.execute("INSERT INTO books VALUES(?,?)",(file_path,0))
        con.commit()
        millis = 0

    f.play()
    f.set_time(millis)

    running = True

    restart = args.restart

    if restart:
        running = False

    w.nodelay(True)
    speed = 1
    while True:
        tag = TinyTag.get(file_path)
        file_str = os.path.basename(file_path)
        file_str = os.path.splitext(file_str)[0]
        total_time_ms = f.get_length()
        w.erase()
        w.border()
        height,width = w.getmaxyx()
        current_time_ms = f.get_time()

        if total_time_ms == 0:
            total_time_ms = 9999999999
        progress = current_time_ms/total_time_ms
        progress_percent = str(round((current_time_ms/total_time_ms)*100,1))
        progress = int(progress*(width-8))
        if progress == 0:
            progress = 1
        empty = (width-14)-progress

        progress_bar = ''

        for c in range(0,progress):
            progress_bar = progress_bar + '█'
        for c in range(0,empty):
            progress_bar = progress_bar + '🮀'

        if running:
            progress_percent = progress_percent+'%'
            w.addstr(2,width-8,progress_percent)
            w.addstr(2,4,progress_bar)
            w.addstr(2,width-10,'▎')
            
            if f.is_playing():
                w.addstr(height-3,4,'SPACE : pause  |  q : exit  |  RIGHT : +15 sec  |  LEFT : -15 sec')
            else:
                w.addstr(2,int(width/2)-6,'   PAUSED   ')
                w.addstr(height-3,4,'SPACE : play   |  q : exit  |  RIGHT : +15 sec  |  LEFT : -15 sec')

            w.addstr(6,4,'Listening to',curses.A_BOLD+curses.color_pair(1))
            if tag.title:
                title_len = len(tag.title)
                w.addstr(6,17,tag.title,curses.A_BOLD+curses.color_pair(2))
                w.addstr(6,17+title_len,' by ',curses.A_BOLD+curses.color_pair(1))
                w.addstr(6,21+title_len,tag.artist,curses.color_pair(3))
            else:
                w.addstr(6,17,file_str,curses.A_BOLD+curses.color_pair(2))
            if tag.comment:
                w.addstr(8,4,tag.comment)
            w.addstr(4,4,' ')
            if f.is_playing():
                w.addstr(4,4,' ')
            w.addstr(4,7,millis_to_timestr(current_time_ms) + 
                         ' / '+
                         millis_to_timestr(total_time_ms) + 
                         ' Speed: ' + str(round(speed,1)))

            # get keys and do thingh
            char = w.getch()

            if speed < 0.5:
                speed = 0.5
            f.set_rate(speed)

            if char == KEY_SPACE:
                f.pause()
            elif char == curses.KEY_RIGHT:
                f.set_time(current_time_ms+15000)
                w.addstr(4,5,'')
            elif char == curses.KEY_LEFT:
                f.set_time(current_time_ms-15000)
                w.addstr(4,5,'')
            elif char == curses.KEY_UP:
                speed = speed + 0.1
            elif char == curses.KEY_DOWN:
                speed = speed - 0.1
            elif char == KEY_Q:
                running = False
            elif char == curses.KEY_MOUSE:
                try:
                    _, mx, my, _, _ = curses.getmouse()
                    if my == 4 and mx == 4:
                        f.pause()
                    if my == 2 and not (mx < 4 or mx > width-10):
                        bar_width = width-12
                        bar_location = mx - 4
                        go_to_ms = (bar_location/bar_width) * total_time_ms
                        f.set_time(int(go_to_ms))
                except:
                    pass
            

        else:
            f.set_pause(1)
            if restart:
                w.addstr(2,4,'Are you sure you would like to start from the beginning? (y/n)',curses.color_pair(3))
                char = w.getch()
                if char == ord('y'):
                    f.set_time(0)
                    f.play()
                    restart = False
                    running = True
                elif char == ord('n'):
                    f.play()
                    running = True
                    restart = False
            else:
                w.addstr(2,4,'Are you sure you would like to exit? (y/n)',curses.color_pair(3))
                w.addstr(3,4,'Your progress will be saved:',curses.color_pair(2))
                w.addstr(5,8,file_str)
                w.addstr(7,8,millis_to_timestr(current_time_ms))
                char = w.getch()
                if char == ord('y'):
                    break
                elif char == ord('n'):
                    running = True

        sleep(0.01)
    
    cur.execute("UPDATE books SET millis = ? where file = ?;",(current_time_ms,file_path))
    con.commit()
    curses.endwin()
    con.close()
    print(f'Location {current_time_ms}ms saved for {file_path}')

curses.wrapper(main)
