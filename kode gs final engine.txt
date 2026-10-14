/**
 * Dashboard Engine - Backend
 */

function doGet(e) {
  return HtmlService.createHtmlOutputFromFile('index')
    .setTitle('Dashboard Engine')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL)
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}

/**
 * Helper cepat mencari sheet berdasarkan GID
 */
function getSheetByGid(ss, gidStr) {
  var sheets = ss.getSheets();
  for (var i = 0; i < sheets.length; i++) {
    if (sheets[i].getSheetId().toString() === String(gidStr)) {
      return sheets[i];
    }
  }
  throw new Error("GID " + gidStr + " tidak ditemukan di dalam file Google Sheet ini.");
}

/**
 * Helper mencari baris kosong pertama berdasarkan kolom acuan dan baris awal data
 */
function getNextRowInCol(sheet, colIndex, startRow) {
  var lastRow = sheet.getLastRow();
  if (lastRow < startRow) return startRow;
  var colValues = sheet.getRange(1, colIndex, lastRow, 1).getValues();
  for (var r = colValues.length - 1; r >= startRow - 1; r--) {
    if (colValues[r][0] !== "" && colValues[r][0] !== null && colValues[r][0] !== undefined) {
      return r + 2;
    }
  }
  return startRow;
}

/**
 * Helper verifikasi hak akses Admin/Supervisor
 */
function isAdminOrSupervisor(divisi) {
  if (!divisi) return false;
  var d = String(divisi).toLowerCase().trim();
  return d.indexOf("admin") !== -1 || d.indexOf("spv") !== -1 || d.indexOf("supervisor") !== -1 || d.indexOf("ho") !== -1 || d.indexOf("all") !== -1;
}

/**
 * Helper Sanitizer Nama File (Tanpa Regex agar 100% aman di Rhino)
 */
function sanitizeFilename(str) {
  var s = String(str || "TOKO");
  var out = "";
  for (var i = 0; i < s.length; i++) {
    var c = s.charAt(i);
    if ((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') || c === '_' || c === '-') {
      out += c;
    } else {
      out += '_';
    }
  }
  return out;
}

/**
 * Helper Upload Foto Base64 ke Google Drive & Mengembalikan URL dengan Fallback Aman
 */
function uploadImageToDrive(base64Data, filename) {
  return uploadImageToDriveCustomFolder(base64Data, filename, null);
}

function uploadImageToDriveCustomFolder(base64Data, filename, folderId) {
  if (!base64Data || base64Data.indexOf('data:image') === -1) {
    return base64Data || "";
  }
  try {
    var splitData = base64Data.split(',');
    var contentType = splitData[0].split(';')[0].split(':')[1];
    var bytes = Utilities.base64Decode(splitData[1]);
    var blob = Utilities.newBlob(bytes, contentType, filename);

    var file = null;
    if (folderId) {
      try {
        var folder = DriveApp.getFolderById(folderId);
        file = folder.createFile(blob);
      } catch (fErr) {
        file = DriveApp.createFile(blob);
      }
    } else {
      file = DriveApp.createFile(blob);
    }

    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
    return file.getUrl();
  } catch (err) {
    Logger.log("Error upload foto: " + err.message);
    return "Error Upload: " + err.message;
  }
}

/**
 * Fetcher tanggal update dengan sistem cache (15 Menit)
 */
function fetchUpdateDate() {
  try {
    var cache = CacheService.getScriptCache();
    var cached = cache.get("SCORECARD_UPDATE_DATE");
    if (cached) return cached;

    var ss = SpreadsheetApp.openById("1N0AB8WHFEOSVlwlaRpwljucGOrBXDGSxnjLjS1s6xgI");
    var sheet = getSheetByGid(ss, "1339176422");
    var val = sheet.getRange("C2").getValue();
    var result = "";
    if (val instanceof Date) {
      result = Utilities.formatDate(val, "GMT+7", "EEEE, dd MMMM yyyy");
    } else {
      result = val ? String(val) : "";
    }
    if (result) cache.put("SCORECARD_UPDATE_DATE", result, 900);
    return result;
  } catch (e) {
    return "";
  }
}

/**
 * Fetcher Master Item List dari Spreadsheet 19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0 GID 1476707678
 * Kolom A = Kode Item, Kolom B = Nama Barang / Item
 */
function getMasterItemData() {
  try {
    var cache = CacheService.getScriptCache();
    var cached = cache.get("MASTER_ITEM_LIST_V1");
    if (cached) {
      try { return JSON.parse(cached); } catch(e) {}
    }

    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "1476707678");
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length < 2) {
      return { success: true, items: [] };
    }

    var items = [];
    for (var r = 1; r < data.length; r++) {
      var kd = data[r][0] !== undefined && data[r][0] !== null ? String(data[r][0]).trim() : "";
      var nm = data[r][1] !== undefined && data[r][1] !== null ? String(data[r][1]).trim() : "";
      if (kd !== "" || nm !== "") {
        items.push({
          kode: kd,
          nama: nm,
          label: kd + (nm ? (" - " + nm) : "")
        });
      }
    }

    var resultObj = { success: true, items: items };
    try {
      var jsonStr = JSON.stringify(resultObj);
      if (jsonStr.length < 95000) {
        cache.put("MASTER_ITEM_LIST_V1", jsonStr, 1800); // Cache 30 Menit
      }
    } catch(err) {}

    return resultObj;
  } catch (err) {
    return { success: false, message: err.message, items: [] };
  }
}

/**
 * Verifikasi Login User & Presensi Otomatis
 */
function verifyLogin(userKey, password) {
  try {
    var ss = SpreadsheetApp.openById("1WuZDbU_gUS-PhS-H_A1iIXEeFqEnQYeT56DiinZQa8Q");
    var sheet = getSheetByGid(ss, "806111467");
    var data = sheet.getDataRange().getValues();

    if (data.length < 2) {
      return { success: false, message: "Data pengguna kosong di server." };
    }

    var inputKey = userKey ? String(userKey).trim().toLowerCase() : "";
    var inputPass = password ? String(password).trim() : "";

    for (var r = 1; r < data.length; r++) {
      var row = data[r];
      var depo = row[0] ? String(row[0]).trim() : "";
      var kdSls = row[1] ? String(row[1]).trim() : "";
      var namaSls = row[2] ? String(row[2]).trim() : "";
      var passVal = row[3] ? String(row[3]).trim() : "";
      var divisi = row[4] ? String(row[4]).trim() : "";

      var matchUser = (kdSls.toLowerCase() === inputKey || namaSls.toLowerCase() === inputKey);
      var matchPass = (passVal === inputPass);

      if (matchUser && matchPass) {
        recordLoginAttendance(sheet, r + 1);

        return {
          success: true,
          user: {
            depo: depo,
            kdSls: kdSls,
            namaSls: namaSls,
            divisi: divisi
          }
        };
      }
    }

    return { success: false, message: "Kode/Nama Salesman atau Password salah!" };

  } catch (err) {
    return { success: false, message: "Terjadi kesalahan: " + err.message };
  }
}

function recordLoginAttendance(sheet, rowIndex) {
  try {
    var now = new Date();
    var day = now.getDate();
    var currentMonthYear = Utilities.formatDate(now, "GMT+7", "yyyy-MM");

    var scriptProps = PropertiesService.getScriptProperties();
    var lastMonth = scriptProps.getProperty('LAST_LOGIN_MONTH');

    if (lastMonth !== currentMonthYear) {
      var lastRow = sheet.getLastRow();
      if (lastRow >= 2) {
        sheet.getRange(2, 7, lastRow - 1, 31).clearContent();
      }
      scriptProps.setProperty('LAST_LOGIN_MONTH', currentMonthYear);
    }

    var colIndex = 6 + day;
    var timeStr = Utilities.formatDate(now, "GMT+7", "HH:mm:ss");
    sheet.getRange(rowIndex, colIndex).setValue(timeStr);
  } catch (e) {
    Logger.log("Error recordLoginAttendance: " + e.toString());
  }
}

function getOmsetRealtimeData() {
  try {
    var ss = SpreadsheetApp.openById("1ssi1n1H2UhSGsQsBG-dV1lcaWchBSKL9FLNrkpluDOM");
    var sheet = getSheetByGid(ss, "1648257239");
    var data = sheet.getDataRange().getValues();

    if (data.length < 1) {
      return { success: true, headers: [], rows: [], lastUpdate: fetchUpdateDate() };
    }

    var rawHeaders = data[0] || [];
    var rawRows = data.slice(1);

    var headers = rawHeaders.slice(1).map(function(h, idx) {
      return (h !== "" && h !== null && h !== undefined) ? String(h).trim() : ("Kolom " + (idx + 2));
    });

    var rows = [];
    rawRows.forEach(function(row, rIdx) {
      var sliceCells = row.slice(1);
      var hasData = sliceCells.some(function(cell) {
        return cell !== "" && cell !== null && cell !== undefined;
      });

      if (hasData) {
        rows.push({
          sheetRow: rIdx + 2,
          cells: sliceCells
        });
      }
    });

    return {
      success: true,
      headers: headers,
      rows: rows,
      lastUpdate: fetchUpdateDate()
    };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function getMonitoringData() {
  try {
    var ss = SpreadsheetApp.openById("1gC5XIbtA5SCrxKTYoEPuYL0Y-RzY5GYjqoyNxJjg64Q");
    var sheet = getSheetByGid(ss, "0");
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length < 2) {
      return { success: true, headers: [], rows: [], lastUpdate: fetchUpdateDate() };
    }

    var headers = data[1] || [];
    var rows = data.slice(2).filter(function(row) {
      return row.some(function(cell) {
        return cell !== "" && cell !== null && cell !== undefined;
      });
    });

    return {
      success: true,
      headers: headers,
      rows: rows,
      lastUpdate: fetchUpdateDate()
    };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function getMonitoringEceranData() {
  try {
    var ss = SpreadsheetApp.openById("1gC5XIbtA5SCrxKTYoEPuYL0Y-RzY5GYjqoyNxJjg64Q");
    var sheet = getSheetByGid(ss, "1538649350");
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length < 2) {
      return { success: true, headers: [], rows: [], lastUpdate: fetchUpdateDate() };
    }

    var headers = data[1] || [];
    var rows = data.slice(2).filter(function(row) {
      return row.some(function(cell) {
        return cell !== "" && cell !== null && cell !== undefined;
      });
    });

    return {
      success: true,
      headers: headers,
      rows: rows,
      lastUpdate: fetchUpdateDate()
    };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function getHeroesSubTabData(gid, headerRowIndex, ssId) {
  try {
    var idx = headerRowIndex !== undefined && headerRowIndex !== null ? Number(headerRowIndex) : 3;
    var targetSsId = ssId || "1aW0Ccr9jujcDtRqgldRrqdtmeN4gw1bM6VVUubyOzsM";

    var ss = SpreadsheetApp.openById(targetSsId);
    var sheet = getSheetByGid(ss, gid);
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length <= idx) {
      return { success: true, criteria: [], headers: [], rows: [], lastUpdate: fetchUpdateDate() };
    }

    var selectedHeaderRow = data[idx] || [];
    var isBadHeader = selectedHeaderRow.some(function(cell) {
      return String(cell).toUpperCase().indexOf("UPDATE") !== -1;
    });

    if (isBadHeader) {
      var bestIdx = idx;
      var maxHeaderCount = 0;
      for (var checkRow = 0; checkRow < Math.min(data.length, 6); checkRow++) {
        var rowText = data[checkRow] || [];
        var countTitles = 0;
        rowText.forEach(function(c) {
          var str = String(c).trim();
          if (str !== "" && str.toUpperCase().indexOf("UPDATE") === -1 && isNaN(Number(str))) {
            countTitles++;
          }
        });
        if (countTitles > maxHeaderCount) {
          maxHeaderCount = countTitles;
          bestIdx = checkRow;
        }
      }
      idx = bestIdx;
    }

    var rawCriteria = (idx >= 1 && data[idx - 1]) ? data[idx - 1] : [];
    var rawHeaders = data[idx] || [];
    var rawRows = data.slice(idx + 1);

    var maxCols = 0;
    for (var r = 0; r < data.length; r++) {
      if (data[r].length > maxCols) maxCols = data[r].length;
    }

    var validIndices = [];
    var headers = [];
    var criteria = [];
    for (var c = 0; c < maxCols; c++) {
      var h = rawHeaders[c];
      var headerStr = (h !== "" && h !== null && h !== undefined) ? String(h).trim() : ("Kolom " + (c + 1));
      validIndices.push(c);
      headers.push(headerStr);
      criteria.push(rawCriteria[c] !== undefined ? String(rawCriteria[c]).trim() : "");
    }

    var rows = [];
    rawRows.forEach(function(row, rIdx) {
      var sheetRowNumber = rIdx + idx + 2;
      var filteredCells = validIndices.map(function(cIdx) {
        return row[cIdx] !== undefined ? row[cIdx] : "";
      });

      var hasData = filteredCells.some(function(cell) {
        return cell !== "" && cell !== null && cell !== undefined;
      });

      if (hasData) {
        rows.push({
          sheetRow: sheetRowNumber,
          cells: filteredCells,
          origCols: validIndices
        });
      }
    });

    return {
      success: true,
      criteria: criteria,
      headers: headers,
      rows: rows,
      lastUpdate: fetchUpdateDate()
    };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function getHeroesNabatiData() {
  return getHeroesSubTabData("248207488", 3);
}

function getScorecardData() {
  try {
    var ss = SpreadsheetApp.openById("1N0AB8WHFEOSVlwlaRpwljucGOrBXDGSxnjLjS1s6xgI");
    var sheet = getSheetByGid(ss, "1339176422");
    var data = sheet.getDataRange().getValues();

    if (data.length < 4) {
      return { success: true, criteria: [], headers: [], rows: [], lastUpdate: fetchUpdateDate() };
    }

    var criteria = data[2] || [];
    var headers = data[3] || [];
    var rows = data.slice(4).filter(function(row) {
      return row.some(function(cell) {
        return cell !== "" && cell !== null && cell !== undefined;
      });
    });

    return {
      success: true,
      criteria: criteria,
      headers: headers,
      rows: rows,
      lastUpdate: fetchUpdateDate()
    };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function getKompetisiData() {
  try {
    var ss = SpreadsheetApp.openById("1O4gIP5d78NmVDWzQ_aGOCoadBx79vgPjsX1uZUR4Tp8");
    var meijiSheet = getSheetByGid(ss, "2127133764");
    var meijiData = meijiSheet.getDataRange().getValues();
    var meijiHeaders = meijiData.length >= 3 ? meijiData[2] : [];
    var meijiRows = meijiData.length > 3 ? meijiData.slice(3).filter(function(row) {
      return row.some(function(cell) { return cell !== "" && cell !== null && cell !== undefined; });
    }) : [];

    var simbaSheet = getSheetByGid(ss, "271515604");
    var simbaData = simbaSheet.getDataRange().getValues();
    var simbaHeaders = simbaData.length >= 3 ? simbaData[2] : [];
    var simbaRows = simbaData.length > 3 ? simbaData.slice(3).filter(function(row) {
      return row.some(function(cell) { return cell !== "" && cell !== null && cell !== undefined; });
    }) : [];

    return {
      success: true,
      meiji: { headers: meijiHeaders, rows: simbaRows },
      simba: { headers: simbaHeaders, rows: simbaRows },
      lastUpdate: fetchUpdateDate()
    };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

/* ==========================================================================
   MENU PENGAJUAN (ABSENSI, BS & CN, NOO, INACTIVE, VALIDASI STATUS)
   ========================================================================== */

function submitAbsensiSalesman(formData, userDivisi) {
  try {
    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "568054987");

    var targetRow = getNextRowInCol(sheet, 2, 6);
    var autoNo = targetRow - 5;

    var values = [
      autoNo,
      formData.tgl || Utilities.formatDate(new Date(), "GMT+7", "yyyy-MM-dd"),
      formData.area || "",
      formData.kdSales || "",
      formData.namaSalesman || "",
      formData.izin || "",
      formData.keterangan || "",
      formData.bukti || ""
    ];

    sheet.getRange(targetRow, 1, 1, values.length).setValues([values]);

    return { success: true, message: "Pengajuan Absensi berhasil disimpan dengan NO: " + autoNo };
  } catch (err) {
    return { success: false, message: "Gagal menyimpan: " + err.message };
  }
}

function getAbsensiSalesmanData() {
  try {
    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "568054987");
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length < 5) {
      return { success: true, headers: [], rows: [] };
    }

    var headers = data[4] || [];
    var rows = data.slice(5).filter(function(row) {
      return row.some(function(cell) { return cell !== "" && cell !== null; });
    });

    rows.reverse();

    return { success: true, headers: headers, rows: rows };
  } catch (err) {
    return { success: false, message: err.message };
  }
}
function submitPengajuanBSCN(formData) {
  try {
    var ssId = '19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0';
    var targetGid = 932209069;

    var ss = SpreadsheetApp.openById(ssId);
    var sheets = ss.getSheets();
    var sheet = null;
    for (var i = 0; i < sheets.length; i++) {
      if (sheets[i].getSheetId() === targetGid) {
        sheet = sheets[i];
        break;
      }
    }

    if (!sheet) {
      return { success: false, message: "Sheet dengan GID " + targetGid + " tidak ditemukan." };
    }

    // CARI BARIS KOSONG BERDASARKAN KOLOM B (Kolom 2)
    var colBValues = sheet.getRange("B:B").getValues();
    var lastRow = 1;
    for (var i = colBValues.length - 1; i >= 0; i--) {
      if (colBValues[i][0] !== "") {
        lastRow = i + 1; // Baris terakhir yang ada isinya di Kolom B
        break;
      }
    }
    var targetRow = lastRow + 1; // Baris kosong berikutnya di Kolom B

    // Tulis data HANYA ke kolom yang diinput manual (Lewati F, H, J, dll.)
    sheet.getRange(targetRow, 1).setValue(formData.tgl);        // Kolom A
    sheet.getRange(targetRow, 2).setValue(formData.tugulCn);    // Kolom B
    sheet.getRange(targetRow, 3).setValue(formData.pma);        // Kolom C
    // Kolom D (Nama Toko) dilompati agar rumus tidak terhapus
    sheet.getRange(targetRow, 5).setValue(formData.kodeToko);   // Kolom E

    // Kolom F (Nama Toko) dilompati agar rumus tidak terhapus

    sheet.getRange(targetRow, 7).setValue(formData.kodeItem);   // Kolom G

    // Kolom H (Nama Item) dilompati agar rumus tidak terhapus

    sheet.getRange(targetRow, 9).setValue(formData.qtyRetur);   // Kolom I

    // Kolom J (Value Retur) dilompati agar rumus tidak terhapus

    sheet.getRange(targetRow, 16).setValue(formData.tglExp);    // Kolom P
    sheet.getRange(targetRow, 17).setValue(formData.issue);     // Kolom Q
    sheet.getRange(targetRow, 18).setValue(formData.reason);    // Kolom R

    return { success: true, message: "Pengajuan BS & CN berhasil dikirim!" };

  } catch (error) {
    return { success: false, message: "Error sistem: " + error.toString() };
  }
}

function getPengajuanBSCNData() {
  try {
    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "932209069");
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length < 3) {
      return { success: true, headers: [], rows: [] };
    }

    var headers = data[2] || [];
    var rows = data.slice(3).filter(function(row) {
      var colAAwal = row[0] !== undefined && row[0] !== null ? String(row[0]).trim() : "";
      if (colAAwal === "") return false;

      return row.some(function(cell) { return cell !== "" && cell !== null; });
    });

    rows.reverse();

    return { success: true, headers: headers, rows: rows };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function submitPengajuanNOO(formData) {
  try {
    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "1906718403");

    var targetRow = getNextRowInCol(sheet, 1, 2);
    var timestamp = Utilities.formatDate(new Date(), "GMT+7", "yyyy-MM-dd HH:mm:ss");

    var timeStampFile = Utilities.formatDate(new Date(), "GMT+7", "yyyyMMdd_HHmmss");
    var tokoClean = sanitizeFilename(formData.namaToko);
    var folderId = "1M_wIIXVO7EUQ2aMscY8bu4TqkyscpMiA";

    var urlFotoDepan = uploadImageToDriveCustomFolder(formData.fotoDepan, "DEPAN_" + tokoClean + "_" + timeStampFile + ".jpg", folderId);
    var urlFotoDalam = uploadImageToDriveCustomFolder(formData.fotoDalam, "DALAM_" + tokoClean + "_" + timeStampFile + ".jpg", folderId);
    var urlFotoKtp   = uploadImageToDriveCustomFolder(formData.fotoKtp,   "KTP_" + tokoClean + "_" + timeStampFile + ".jpg", folderId);

    var values = [
      timestamp,
      formData.depo || "",
      formData.namaSales || "",
      formData.namaToko || "",
      formData.namaPemilik || "",
      formData.titikGps || "",
      formData.alamat || "",
      formData.kelurahan || "",
      formData.kecamatan || "",
      formData.kabupaten || "",
      formData.rayon || "",
      formData.cycle || "",
      formData.typeOutlet || "",
      urlFotoDepan,
      urlFotoDalam,
      urlFotoKtp,
      "",
      "MENUNGGU VALIDASI"
    ];

    sheet.getRange(targetRow, 1, 1, values.length).setValues([values]);

    return { success: true, message: "Pengajuan NOO berhasil dikirim & foto tersimpan!" };
  } catch (err) {
    return { success: false, message: "Gagal menyimpan pengajuan NOO: " + err.message };
  }
}

function getPengajuanNOOData() {
  try {
    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "1906718403");
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length < 1) {
      return { success: true, headers: [], rows: [] };
    }

    var headers = data[0] || [];
    var rows = [];
    for (var r = 1; r < data.length; r++) {
      var row = data[r];
      if (row.some(function(cell) { return cell !== "" && cell !== null; })) {
        rows.push({
          sheetRow: r + 1,
          cells: row
        });
      }
    }

    return { success: true, headers: headers, rows: rows };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function submitPengajuanInactive(formData) {
  try {
    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "0");

    var targetRow = getNextRowInCol(sheet, 1, 2);
    var timestamp = Utilities.formatDate(new Date(), "GMT+7", "yyyy-MM-dd HH:mm:ss");

    var timeStampFile = Utilities.formatDate(new Date(), "GMT+7", "yyyyMMdd_HHmmss");
    var tokoClean = sanitizeFilename(formData.namaToko);

    var urlFotoToko = uploadImageToDrive(formData.fotoToko, "INACTIVE_" + tokoClean + "_" + timeStampFile + ".jpg");

    var values = [
      timestamp,
      formData.depo || "",
      formData.kdSales || "",
      formData.namaSalesman || "",
      formData.kdToko || "",
      formData.namaToko || "",
      formData.titikGps || "",
      formData.alamat || "",
      formData.kelurahan || "",
      formData.kecamatan || "",
      formData.kabupaten || "",
      formData.alasanInactive || "",
      urlFotoToko,
      "MENUNGGU VALIDASI"
    ];

    sheet.getRange(targetRow, 1, 1, values.length).setValues([values]);

    return { success: true, message: "Pengajuan Inactive Toko berhasil dikirim & Foto tersimpan!" };
  } catch (err) {
    return { success: false, message: "Gagal menyimpan: " + err.message };
  }
}

function getPengajuanInactiveData() {
  try {
    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var sheet = getSheetByGid(ss, "0");
    var data = sheet.getDataRange().getDisplayValues();

    if (data.length < 1) {
      return { success: true, headers: [], rows: [] };
    }

    var headers = data[0] || [];
    var rows = [];
    for (var r = 1; r < data.length; r++) {
      var row = data[r];
      if (row.some(function(cell) { return cell !== "" && cell !== null; })) {
        rows.push({
          sheetRow: r + 1,
          cells: row
        });
      }
    }

    return { success: true, headers: headers, rows: rows };
  } catch (err) {
    return { success: false, message: err.message };
  }
}

function updateStatusPengajuan(type, sheetRow, newStatus, userDivisi) {
  try {
    if (!isAdminOrSupervisor(userDivisi)) {
      return { success: false, message: "Akses Ditolak! Fitur ini khusus Admin atau Supervisor." };
    }

    var ss = SpreadsheetApp.openById("19ubFPaIp1ZPeUukWUkarl0lwHfabsQnoNQnqOr9nyL0");
    var gid = (type === 'noo') ? "1906718403" : "0";
    var statusCol = (type === 'noo') ? 18 : 14;

    var sheet = getSheetByGid(ss, gid);
    sheet.getRange(sheetRow, statusCol).setValue(newStatus);

    return { success: true, message: "Status " + type.toUpperCase() + " berhasil diperbarui menjadi: " + newStatus };
  } catch (err) {
    return { success: false, message: "Gagal memperbarui status: " + err.message };
  }
}
