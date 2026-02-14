/******************** CONFIG ********************/
const SPREADSHEET_ID = '1yF2jIPcKXgGLDQFcsDABY-zQeoj87XquGaENwTgN8xQ';
const ROOT_FOLDER_NAME = 'AnimalSurvey_OwnerPhotos';

/******************** MASTER HEADER ********************/
const MASTER_HEADERS = [
"timestamp","ปีงบประมาณ","ศูนย์บริการ","ชุมชน",
"ประเภทสัตว์","ชนิดสัตว์ (อื่น)","จำนวนสัตว์ (อื่น)",
"ลำดับสัตว์","ชื่อสัตว์","เพศ","อายุ (ปี)","อายุ (เดือน)",
"สี / ตำหนิ","สถานะทำหมัน","วันที่ฉีดยาคุม","สัตวแพทย์ผู้ฉีดยาคุม",
"สถานะวัคซีนพิษสุนัขบ้า","วันที่ฉีดวัคซีน","สัตวแพทย์ผู้ฉีดวัคซีน",
"ลักษณะการเลี้ยง","สถานที่เลี้ยง","พื้นที่การเลี้ยง",
"ชื่อเจ้าของสัตว์","เลขบัตรประชาชน","เบอร์โทรศัพท์มือถือ",
"เบอร์โทรศัพท์บ้าน","บ้านเลขที่","ถนน","ซอย",
"ตำบล","อำเภอ","จังหวัด",
"ผู้บันทึก","ตำแหน่ง",
"ลิงก์รูปเจ้าของ","ลิงก์รูปสัตว์"
];

/******************** MAIN ********************/
function doPost(e){

const lock = LockService.getScriptLock();
if(!lock.tryLock(10000))
  return output_('ระบบกำลังประมวลผล กรุณาลองใหม่');

try{

if(!e || !e.postData || !e.postData.contents)
  return output_('No POST data');

let data={};
try{
  data = JSON.parse(e.postData.contents);
}catch(err){
  return output_('JSON ไม่ถูกต้อง');
}

const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
const timestamp = new Date();

const year = safe_(data.year) || '2569';
const centerName = safe_(data.centerName || data.center);
const communityName = safe_(data.communityName || data.community);

if(!centerName || !communityName)
  return output_('กรุณาระบุชื่อศูนย์บริการและชุมชน');

const addr = data.address || {};

/************ PET STRUCTURE ************/
let pets = [];

if(Array.isArray(data.pets) && data.pets.length>0){
  pets = data.pets;
}
else if(data.pet){
  pets = [data.pet];
}
else{
  pets = [data];
}

/************ SHEETS ************/
const centerSheet = getOrCreateSheet_(ss, centerName);
const communitySheet = getOrCreateSheet_(ss, communityName);

ensureHeaders_(centerSheet);
ensureHeaders_(communitySheet);

/************ DRIVE ************/
const rootFolder = getOrCreateFolder_(null, ROOT_FOLDER_NAME);
const yearFolder = getOrCreateFolder_(rootFolder, year);
const centerFolder = getOrCreateFolder_(yearFolder, cleanName_(centerName));
const communityFolder = getOrCreateFolder_(centerFolder, cleanName_(communityName));

/************ OWNER FOLDER FORMAT ************/
const ownerNameClean = cleanName_(safe_(data.ownerName)||'ไม่ระบุ');
const addrNoClean = cleanName_(addr.addrNo || '');
const roadClean = cleanName_(addr.road || '');

let ownerFolderName = 'เจ้าของ_' + ownerNameClean;
if(addrNoClean) ownerFolderName += '_บ้านเลขที่' + addrNoClean;
if(roadClean) ownerFolderName += '_ถนน' + roadClean;

const ownerFolder = getOrCreateFolder_(communityFolder,ownerFolderName);

const ownerPhotoUrl = savePhotoSmart_(
data.ownerPhotoBase64 || data.ownerPhoto || data.ownerImage,
ownerFolder,'รูปเจ้าของ'
);

const petMainFolder = getOrCreateFolder_(ownerFolder,'สัตว์เลี้ยง');

/************ LOOP PET ************/
pets.forEach((pet,index)=>{

let petPhotoUrl='';

/* รองรับ BASE64 */
const petPhotoData =
pet.petPhotoBase64 || pet.photoBase64 ||
pet.image || pet.photo || '';

if(petPhotoData){
  petPhotoUrl = savePhotoSmart_(
    petPhotoData,
    petMainFolder,
    'สัตว์_'+(pet.no||(index+1))+'_'+new Date().getTime()
  );
}

/* รองรับ URL */
if(!petPhotoUrl && pet.imageUrl){
  petPhotoUrl = saveImageFromUrl_(
    pet.imageUrl,
    petMainFolder,
    'สัตว์_'+(index+1)
  );
}

const rowObj = buildRowObject_(
timestamp,year,centerName,communityName,
data,pet,index,addr,
ownerPhotoUrl,petPhotoUrl
);

appendSafe_(centerSheet,rowObj);
appendSafe_(communitySheet,rowObj);

});

return output_('success',true);

}catch(err){
Logger.log(err);
return output_(err.message);
}
finally{
lock.releaseLock();
}
}

/******************** BUILD ROW ********************/
function buildRowObject_(
timestamp,year,centerName,communityName,
data,pet,index,addr,
ownerPhotoUrl,petPhotoUrl){

function pick_(){
for(let i=0;i<arguments.length;i++){
const v=arguments[i];
if(v!==undefined&&v!==null&&v!=='')
return v.toString().trim();
}
return '';
}

/************ ประเภทสัตว์ ************/
let rawType = pick_(pet.animalType,data.animalType);
let otherType = pick_(pet.otherAnimalType,data.otherAnimalType);
let animalType='ไม่ระบุ';

if(rawType){
  const t=rawType.trim();
  if(['สุนัข','หมา','dog'].includes(t)) animalType='สุนัข';
  else if(['แมว','cat'].includes(t)) animalType='แมว';
  else if(['เป็ด'].includes(t)) animalType='เป็ด';
  else if(['ไก่'].includes(t)) animalType='ไก่';
  else if(['สุกร','หมู'].includes(t)) animalType='สุกร';
  else if(['นก'].includes(t)) animalType='นก';
  else if(['แพะ'].includes(t)) animalType='แพะ';
  else if(['ม้า'].includes(t)) animalType='ม้า';
  else if(['อื่นๆ'] .includes(t) && otherType)
       animalType='อื่นๆ ('+otherType+')';
  else animalType=t;
}

/************ ลักษณะการเลี้ยง ************/
let raisingRaw = pick_(
pet.raisingType,pet.raising,pet.raising_type,
data.raisingType,data.raising,data.raising_type
);

let raisingType='ไม่ระบุ';

if(raisingRaw==='1') raisingType='เลี้ยงในพื้นที่จำกัดตลอดเวลา';
else if(raisingRaw==='2') raisingType='เลี้ยงแบบปล่อยตลอดเวลา';
else if(raisingRaw==='3') raisingType='เลี้ยงในพื้นที่จำกัดบ้างเวลา';
else if(raisingRaw) raisingType=raisingRaw;

/************ วัคซีนพิษสุนัขบ้า ************/
let rabiesStatus = pick_(
pet.rabiesStatus,pet.rabies,data.rabiesStatus
)||'ไม่ระบุ';

return {

"timestamp":timestamp,
"ปีงบประมาณ":year,
"ศูนย์บริการ":centerName,
"ชุมชน":communityName,

"ประเภทสัตว์":animalType,
"ชนิดสัตว์ (อื่น)":otherType||'',
"จำนวนสัตว์ (อื่น)":safe_(data.otherAnimalQty),

"ลำดับสัตว์":pick_(pet.no,(index+1)),
"ชื่อสัตว์":pick_(pet.name)||'ไม่ระบุ',
"เพศ":pick_(pet.gender)||'ไม่ระบุ',
"อายุ (ปี)":pick_(pet.ageYear),
"อายุ (เดือน)":pick_(pet.ageMonth),
"สี / ตำหนิ":pick_(pet.color),

"สถานะทำหมัน":pick_(pet.sterilization)||'ไม่ระบุ',
"วันที่ฉีดยาคุม":pick_(pet.contraceptiveDate),
"สัตวแพทย์ผู้ฉีดยาคุม":pick_(pet.contraceptiveVet),

"สถานะวัคซีนพิษสุนัขบ้า":rabiesStatus,
"วันที่ฉีดวัคซีน":pick_(pet.rabiesDate),
"สัตวแพทย์ผู้ฉีดวัคซีน":pick_(pet.rabiesVet),

"ลักษณะการเลี้ยง":raisingType,
"สถานที่เลี้ยง":pick_(pet.raisingLocation,data.raisingLocation),
"พื้นที่การเลี้ยง":pick_(pet.raisingArea,data.raisingArea),

"ชื่อเจ้าของสัตว์":pick_(data.ownerName)||'ไม่ระบุ',
"เลขบัตรประชาชน":pick_(data.citizenId),
"เบอร์โทรศัพท์มือถือ":pick_(data.phone),
"เบอร์โทรศัพท์บ้าน":pick_(data.homePhone),

"บ้านเลขที่":pick_(addr.addrNo),
"ถนน":pick_(addr.road),
"ซอย":pick_(addr.soi),
"ตำบล":pick_(addr.subdistrict),
"อำเภอ":pick_(addr.district),
"จังหวัด":pick_(addr.province),

"ผู้บันทึก":pick_(data.recorderName),
"ตำแหน่ง":pick_(data.recorderRole),

"ลิงก์รูปเจ้าของ":ownerPhotoUrl||'',
"ลิงก์รูปสัตว์":petPhotoUrl||''
};
}

/******************** APPEND ********************/
function appendSafe_(sheet,rowObj){
autoAddMissingColumns_(sheet,rowObj);
const headers = sheet.getRange(1,1,1,sheet.getLastColumn())
.getValues()[0].map(h=>h.toString().trim());
const row = headers.map(h=>rowObj[h]||'');
if(row.join('').trim()!=='')
sheet.appendRow(row);
}

function ensureHeaders_(sheet){
if(sheet.getLastRow()===0)
sheet.appendRow(MASTER_HEADERS);
}

function autoAddMissingColumns_(sheet,rowObj){
let headers = sheet.getRange(1,1,1,sheet.getLastColumn())
.getValues()[0].map(h=>h.toString().trim());
Object.keys(rowObj).forEach(key=>{
if(headers.indexOf(key)===-1){
sheet.getRange(1,headers.length+1).setValue(key);
headers.push(key);
}
});
}

/******************** HELPERS ********************/
function safe_(v){
return (v===null||v===undefined)?'':v.toString().trim();
}

function getOrCreateSheet_(ss,name){
let s=ss.getSheetByName(name);
if(!s)s=ss.insertSheet(name);
return s;
}

function getOrCreateFolder_(parent,name){
name=(name||'Unknown').toString().trim();
if(!parent){
const it=DriveApp.getFoldersByName(name);
return it.hasNext()?it.next():DriveApp.createFolder(name);
}
const it=parent.getFoldersByName(name);
return it.hasNext()?it.next():parent.createFolder(name);
}

function cleanName_(text){
return (text||'').toString().trim()
.replace(/\s+/g,'_')
.replace(/[\\/:*?"<>|]/g,'');
}

function savePhotoSmart_(base64Data,folder,fileName){
if(!base64Data||typeof base64Data!=='string')
return '';
try{
if(base64Data.indexOf('base64,')!==-1)
base64Data=base64Data.split('base64,')[1];
const bytes=Utilities.base64Decode(base64Data);
const blob=Utilities.newBlob(bytes,'image/jpeg',fileName+'.jpg');
const file=folder.createFile(blob);
file.setSharing(DriveApp.Access.ANYONE_WITH_LINK,DriveApp.Permission.VIEW);
return file.getUrl();
}catch(err){
Logger.log(err);
return '';
}
}

function saveImageFromUrl_(url,folder,fileName){
try{
const response = UrlFetchApp.fetch(url);
const blob = response.getBlob();
const file = folder.createFile(blob).setName(fileName+'.jpg');
file.setSharing(DriveApp.Access.ANYONE_WITH_LINK,DriveApp.Permission.VIEW);
return file.getUrl();
}catch(e){
Logger.log(e);
return '';
}
}

function output_(msg,ok){
return ContentService.createTextOutput(
JSON.stringify({status:ok?'success':'error',message:msg})
).setMimeType(ContentService.MimeType.JSON);
}
