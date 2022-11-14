from flask import Flask, app, request, render_template
import os
import flask
import re
import flask_login
import base64
from PIL import Image
from io import BytesIO
import datetime

from cloudant.client import Cloudant
from cloudant.error import CloudantException
from cloudant.result import Result, ResultByKey



client = Cloudant.iam(
    '6803cce5-6842-45c7-b5e2-65a08f2d80fa-bluemix','reiQA6VSK7w6yG9lKhhn5Vt-lAWAzrrzZTV4ZnqNoaA_',connect=True)
name = 'name'
email = 'a@b.c'
password = '123'

user_database = client.create_database('user_database')
user_image_database = client.create_database('user_image_database')

def image_database_updation(name,email,imagestr):
    global user_image_database
    json_image_document={
        'name':name,
        'email':email,
        'image':imagestr,
        'datetime':datetime.datetime.now().timestamp() * 1000
    }
    new_image_document  = user_image_database.create_document(json_image_document)
    if(new_image_document.exists()):
        print('database updated')
    else:
        print('database couldn\'t be edited')
    return

def image_database_retrieval():
    global user_image_database
    image_result_retrieved = Result(user_image_database.all_docs,include_docs=True)
    image_result ={}
    for i in image_result_retrieved:
        image_result[i['doc']['email']] = {'name':i['doc']['name'],'image':i['doc']['image'],'date':i['doc']['datetime']}
    return(image_result)

def database_updation(name,email,password):
    global user_database
    jsonDocument = {
        'name':name,
        'email':email,
        'password':password
    }
    newDocument = user_database.create_document(jsonDocument)
    if(newDocument.exists()):
        print('database updated')
    else:
        print('database couldn\'t be edited')
    return
#database_updation(name,email,password)

def database_retrieval():
    global user_database
    result_retrieved = Result(user_database.all_docs,include_docs=True)
    #print(list(result_retrieved))
    result = {}
    for i in list(result_retrieved):
        result[i['doc']['email']]={'name':i['doc']['name'],'password':i['doc']['password']}
    return result
#print(database_retrieval())
app = Flask(__name__)
app.secret_key = 'apple'
login_manager = flask_login.LoginManager()

login_manager.init_app(app)
users = {'a@b.c': {'password': '123'}}
class User(flask_login.UserMixin):
    pass


@login_manager.user_loader
def user_loader(email):
    data = database_retrieval()
    if email not in data:
        return

    user = User()
    user.id = email
    user.name = data[email]['name']
    return user


@login_manager.request_loader
def request_loader(request):
    email = request.form.get('email')
    data = database_retrieval()
    if email not in data:
        return

    user = User()
    user.id = email
    user.name = data[email]['name']
    return user
@app.route('/')
def index():
    if(flask_login.current_user.is_authenticated):
        return render_template('dashboard.html')
    else:
        return render_template('login.html')

@app.route('/register',methods = ['GET','POST'])
def register():
    data = database_retrieval()
    if(flask.request.method == 'GET'):
        return render_template('register.html')
    email = flask.request.form['email']
    if(email in data):
        return render_template('register.html',flash_message='True')
    else:
        database_updation(flask.request.form['name'],email,flask.request.form['password'])
        #users[email]={'password':flask.request.form['password']}
        user  = User()
        user.id = email
        user.name = flask.request.form['name']
        flask_login.login_user(user)
        return render_template('dashboard.html',flash_message='True')



@app.route('/login',methods =['GET','POST'])
def login():
    data = database_retrieval()
    if(flask.request.method == 'GET'):
        
        return render_template('login.html',flash_message='False')
    email = flask.request.form['email']
    if(email in data and flask.request.form['password']==data[email]['password']):
        user  = User()
        user.id = email
        flask_login.login_user(user)
        return render_template('dashboard.html',flash_message='Fal')
    #flask.flash('invalid credentials !!!')
    return render_template('login.html',flash_message="True")
    #error = 'inavlid credentials')


@app.route('/dashboard',methods = ['GET','POST'])
@flask_login.login_required
def dashboard():
    if(flask.request.method == 'GET'):
        return render_template('dashboard.html',flash_message='False')
    email = flask.request.form['email']
    if(email in users and flask.request.form['password']==users[email]['password']):
        user  = User()
        user.id = email
        flask_login.login_user(user)
        return render_template('dashboard.html',flash_message="Fal")
    return render_template('dashboard.html',flash_message="Fals")


@app.route('/logout')
@flask_login.login_required
def logout():
    flask_login.logout_user()
    return render_template('logout.html')


@app.route('/prediction',methods = ['GET','POST'])
@flask_login.login_required
def prediction():
    if(flask.request.method=='POST'):
        img = flask.request.files['myFile']
        try:
            os.remove('Project Development Phase\static\imagedata\save.png')
        except:
            pass
        imgstr = base64.b64encode(img.read()).decode('utf-8')
        image_database_updation(flask_login.current_user.name,flask_login.current_user.id,imgstr)
        data = image_database_retrieval()
        print(flask_login.current_user.id)
        #data = dict(sorted(data.items(),key=lambda item:item['date']))
        img_retrived = Image.open(BytesIO(base64.b64decode(data[flask_login.current_user.id]['image'])))
        img_retrived.save('Project Development Phase\static\imagedata\save.png')
        print('image uploaded and retrieved')
        return render_template('prediction.html',flash_message='True')
        #,imag=img_retrived)

    return render_template('prediction.html')

if __name__ == '__main__':
    app.run(debug=True,port = 8000)