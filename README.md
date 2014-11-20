upload-image-from-database-
===========================
Image  downloding 


import req_proxy
from bs4 import BeautifulSoup
from lxml import html
import re
import time
import logging
import sys
import ast
import urllib
import urllib2
import os
import boto
from boto.s3.key import Key
import shutil
import MySQLdb

import multiprocessing
import os
from threading import Thread
from Queue import Queue

#logging.basicConfig(level=logging.DEBUG, format='[%(levelname)s] (%(threadName)-10s) %(message)s', )

num_fetch_process = 10
enclosure_queue = Queue()


def to_cdn(filename, keyname):
    #try:
        #boto.set_stream_logger('boto')
        c = boto.connect_s3()
        b = c.get_bucket("zovon")
        k = Key(b)
        k.key = "prod_img/%s" %(keyname)
        k.set_metadata("Content-Type", 'image/jpeg')
        k.set_contents_from_filename(filename)
        k.make_public()
        c.close()
        os.remove(filename)
        
    #except:
     #   pass




def main(i, directory,q):
        transfer_direc = "/home/desktop/anit/zivame/img_not_transfer"
        try:
            os.mkdir(transfer_direc)

        except:
            pass

        for sku, link1 in iter(q.get, None):
    
            try:
                product_id = sku
                link = str(link1).strip()
                     

                page= req_proxy.main(link)
    
                name = link.split("/")[-1].strip()
    
                filename = "%s/%s" %(directory,name)
                f = open(filename,'wb')
                f.write(page)
                f.close()

                filename = str(filename).strip()
                keyname = filter(None, filename.split("/"))[-1]

                to_cdn(filename, keyname)
              #  sqlentry =""" update zivame_images set upload_image_status='Yes' where image_link ="%s" """
               # sqlentry = sqlentry %(link)                
                #cursor.execute(sqlentry)
                #db.commit()
                
                print "<............GOING TO ZOVON..................>"
            except:
                filename = "%s/not_transfer_link.txt" %(transfer_direc)
                f = open(filename,'a+')
                f.write(str([sku, link]) + "\n")
                f.close() 
                


            time.sleep(2)
            q.task_done()

        q.task_done()

def supermain():

    directory = "/home/desktop/anit/zivame/image_folder"
    try:
     os.mkdir(directory)

    except:
        pass


    db = MySQLdb.connect("192.99.13.229","root","6Tresxcvbhy","zivame")
    cursor = db.cursor()

    sql_select = """select product_id,image_link from zivame_images where upload_image_status='NO' """
    cursor.execute(sql_select)

    results = cursor.fetchall()
    result =map(list,results)
   # for line in result:
    #    link = line[1]
     #   sku=line[0]
   
 
    
    proc = []

    for i in range(num_fetch_process):
        proc.append(Thread(name = str(i), target= main, args=(i, directory,enclosure_queue,)))
        proc[-1].start()

    for line in result:
        #print line
        link = line[1]
        sku=line[0]
       # print link
       # print sku

        enclosure_queue.put((sku, link))

    enclosure_queue.join()

    for p in proc:
        enclosure_queue.put(None)

    enclosure_queue.join()

    for p in proc:
        p.join(200)
 
    try:
        shutil.rmtree('/home/desktop/anit/zivame/image_folder')

    except:
        pass
    
if __name__=="__main__":
    #line = ("1", "http://cdn.zivame.com/media/catalog/product/cache/1/image/410x/040ec09b1e35df139433887a97daa66f/c/a/cake_vanilla_cream_bikini_brief.jpg")
    #directory = "/home/desktop/anit/zivame/image_folder" 
    #main(directory, line)
    supermain()
